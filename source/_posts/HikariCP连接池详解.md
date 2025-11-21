---
title: HikariCP 连接池详解
author: Sitoi
top: false
cover: false
mathjax: false
toc: true
abbrlink: 101
date: 2025-10-08 10:30:07
categories:
tags:
img:
coverImg:
password:
summary:
keywords:
---

# HikariCP 连接池详解

## 目录

1. [核心概念](#核心概念)
2. [连接获取流程](#连接获取流程)
3. [连接归还流程](#连接归还流程)
4. [ConcurrentBag 三层查找机制](#concurrentbag-三层查找机制)
5. [关键组件详解](#关键组件详解)
6. [常见问题解答](#常见问题解答)
7. [设计决策分析](#设计决策分析)

---

## 核心概念

### 1. 基本组件

#### Connection（真实连接）
- 数据库驱动提供的真实 JDBC 连接
- 例如 MySQL 的 `com.mysql.cj.jdbc.ConnectionImpl`
- 直接与数据库通信

#### PoolEntry（连接条目）
- 连接池内部用来管理连接的包装类
- 每个 `PoolEntry` 唯一对应一个真实的 `Connection`
- 包含连接的状态、时间戳等信息

```java
final class PoolEntry implements IConcurrentBagEntry {
    Connection connection;      // 真实连接
    long lastAccessed;          // 最后访问时间
    long lastBorrowed;          // 最后借出时间
    private volatile int state; // 状态（空闲/使用中/已移除等）
    private volatile boolean evict; // 是否被标记驱逐
}
```

#### ProxyConnection（代理连接）
- 应用层拿到的连接对象
- 内部持有 `PoolEntry` 和真实 `Connection`
- 拦截 `close()` 等方法，将连接归还池中而不是真正关闭

#### ConcurrentBag（连接袋）
- 存放 `PoolEntry` 的容器
- 提供 `borrow()` 和 `requite()` 方法
- 使用线程本地缓存和共享列表，减少锁竞争

#### bagEntry
- `bagEntry` 就是 `PoolEntry`
- 在 `ConcurrentBag` 的方法里，参数名用 `bagEntry` 表示"袋子里的条目"

### 2. 关系图

```
应用代码
   ↓ 调用 getConnection()
HikariPool
   ↓ 从 ConcurrentBag 借出
PoolEntry (bagEntry)
   ↓ 包装成
ProxyConnection
   ↓ 返回给应用
应用拿到 ProxyConnection，但实际使用的是 PoolEntry 里的真实 Connection
```

---

## 连接获取流程

### 1. 入口方法

```java
public Connection getConnection(final long hardTimeout) throws SQLException {
    suspendResumeLock.acquire();
    final long startTime = currentTime();
    
    try {
        long timeout = hardTimeout;
        do {
            // 从 ConcurrentBag 借出 PoolEntry
            PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
            if (poolEntry == null) {
                break; // 超时
            }
            
            final long now = currentTime();
            // 健康检查
            if (poolEntry.isMarkedEvicted() || 
                (elapsedMillis(poolEntry.lastAccessed, now) > aliveBypassWindowMs 
                 && !isConnectionAlive(poolEntry.connection))) {
                // 连接不可用，关闭并重试
                closeConnection(poolEntry, ...);
                timeout = hardTimeout - elapsedMillis(startTime);
            } else {
                // 连接可用，创建代理连接返回
                metricsTracker.recordBorrowStats(poolEntry, startTime);
                return poolEntry.createProxyConnection(
                    leakTaskFactory.schedule(poolEntry), now);
            }
        } while (timeout > 0L);
        
        // 超时，抛出异常
        metricsTracker.recordBorrowTimeoutStats(startTime);
        throw createTimeoutException(startTime);
    } finally {
        suspendResumeLock.release();
    }
}
```

### 2. 关键步骤

1. **获取 suspendResumeLock**：确保池未被挂起
2. **从 ConcurrentBag 借出**：三层查找机制（见下文）
3. **健康检查**：
   - 检查是否被标记驱逐
   - 检查连接是否存活（超过 `aliveBypassWindowMs` 需要检测）
4. **创建代理连接**：包装成 `ProxyConnection` 并启动泄漏检测
5. **返回给应用**：应用获得代理连接

---

## 连接归还流程

### 1. 应用调用 close()

```java
connection.close(); // 应用代码
```

### 2. ProxyConnection 拦截 close()

```java
@Override
public final void close() throws SQLException {
    // 1. 关闭所有 Statement
    closeStatements();
    
    if (delegate != ClosedConnection.CLOSED_CONNECTION) {
        // 2. 取消泄漏检测任务
        leakTask.cancel();
        
        try {
            // 3. 清理连接状态
            if (isCommitStateDirty && !isAutoCommit) {
                delegate.rollback();
            }
            if (dirtyBits != 0) {
                poolEntry.resetConnectionState(this, dirtyBits);
            }
            delegate.clearWarnings();
        } catch (SQLException e) {
            if (!poolEntry.isMarkedEvicted()) {
                throw checkException(e);
            }
        } finally {
            // 4. 归还连接
            delegate = ClosedConnection.CLOSED_CONNECTION;
            poolEntry.recycle(lastAccess);
        }
    }
}
```

### 3. 调用链

```
ProxyConnection.close()
    ↓ 调用 poolEntry.recycle(lastAccess)
PoolEntry.recycle()
    ↓ 调用 hikariPool.recycle(this)
HikariPool.recycle()
    ↓ 调用 connectionBag.requite(poolEntry)
ConcurrentBag.requite()
    ↓ 执行 threadLocalList.add(bagEntry) 或放入 handoffQueue
连接被归还到池中 ✓
```

### 4. requite() 方法详解

```java
public void requite(final T bagEntry) {
    // 1. 设置状态为空闲
    bagEntry.setState(STATE_NOT_IN_USE);
    
    // 2. 如果有等待线程，优先放入 handoffQueue 直接交付
    for (int i = 0; waiters.get() > 0; i++) {
        if (bagEntry.getState() != STATE_NOT_IN_USE || 
            handoffQueue.offer(bagEntry)) {
            return;
        }
        // 重试逻辑...
    }
    
    // 3. 否则放入线程本地列表（最多50个）
    final List<Object> threadLocalList = threadList.get();
    if (threadLocalList.size() < 50) {
        threadLocalList.add(weakThreadLocals ? 
            new WeakReference<>(bagEntry) : bagEntry);
    }
}
```

---

## ConcurrentBag 三层查找机制

### 第一层：线程本地列表（threadList）

#### 工作原理

```java
// Try the thread-local list first
final List<Object> list = threadList.get();
for (int i = list.size() - 1; i >= 0; i--) {
    final Object entry = list.remove(i);
    final T bagEntry = weakThreadLocals ? 
        ((WeakReference<T>) entry).get() : (T) entry;
    if (bagEntry != null && 
        bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
        return bagEntry;
    }
}
```

#### 特点
- **ThreadLocal 存储**：每个线程有独立的列表，互不干扰
- **从后往前遍历**：优先取最近归还的连接
- **CAS 状态切换**：使用 `compareAndSet` 原子操作
- **无锁设计**：线程只访问自己的列表，无需同步

#### 为什么优先使用？
- ✅ 无锁：线程只访问自己的列表，无需同步
- ✅ 缓存局部性：CPU 缓存命中率高
- ✅ 减少竞争：避免多线程争用共享资源

#### 为什么是 List 而不是单个连接？
虽然大多数情况下线程只需要一个连接，但：
1. **连接复用缓存**：归还后可能很快又要用，缓存起来快速获取
2. **支持嵌套场景**：一个线程可能需要多个连接（嵌套事务、并行查询）
3. **限制在50个**：防止无限增长

### 第二层：共享列表（sharedList）

#### 工作原理

```java
// Otherwise, scan the shared list
final int waiting = waiters.incrementAndGet();
try {
    for (T bagEntry : sharedList) {
        if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            // 如果等待线程数 > 1，触发补偿创建
            if (waiting > 1) {
                listener.addBagItem(waiting - 1);
            }
            return bagEntry;
        }
    }
    
    // 没有可用连接，触发创建新连接
    listener.addBagItem(waiting);
    // ...
} finally {
    waiters.decrementAndGet();
}
```

#### 特点
- **共享资源**：所有线程共享的 `CopyOnWriteArrayList`
- **线性扫描**：遍历查找空闲连接
- **CAS 竞争**：多个线程可能同时尝试获取同一连接
- **补偿机制**：如果等待线程数 > 1，触发创建新连接

#### 关键变量
- **sharedList**：存储所有连接的全局列表
- **waiters**：统计当前等待连接的线程数
- **listener**：回调接口，当需要创建新连接时通知外部

### 第三层：等待队列（handoffQueue）

#### 工作原理

```java
// 阻塞等待新连接
timeout = timeUnit.toNanos(timeout);
do {
    final long start = currentTime();
    final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
    if (bagEntry == null || 
        bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
        return bagEntry;
    }
    timeout -= elapsedNanos(start);
} while (timeout > 10_000);
```

#### 特点
- **SynchronousQueue**：无缓冲队列，生产者必须等待消费者
- **阻塞等待**：在超时内等待新连接
- **直接交付**：连接归还时如果有等待线程，直接放入队列

#### 为什么需要？
- 当前无可用连接时，避免忙等待
- 连接归还时直接交付，减少唤醒延迟

---

## 关键组件详解

### 1. aliveBypassWindowMs（健康检查窗口）

```java
private final long aliveBypassWindowMs = Long.getLong(
    "com.zaxxer.hikari.aliveBypassWindowMs", 
    MILLISECONDS.toMillis(500));
```

#### 作用
- 默认 500ms 的时间窗口
- 在这个窗口内再次借出同一连接时，可以跳过昂贵的 `isConnectionAlive` 检查
- 超过窗口时间，需要重新检测连接是否存活

#### 判断逻辑

```java
if (elapsedMillis(poolEntry.lastAccessed, now) > aliveBypassWindowMs 
    && !isConnectionAlive(poolEntry.connection)) {
    // 连接可能已失效，关闭并重试
    closeConnection(poolEntry, DEAD_CONNECTION_MESSAGE);
}
```

### 2. isMarkedEvicted()（驱逐标记）

```java
boolean isMarkedEvicted() {
    return evict;
}

void markEvicted() {
    this.evict = true;
}
```

#### 作用
- 标记连接需要被驱逐
- 当连接被标记后，下次借出时会立即关闭

#### 连接被淘汰的原因
1. **生命周期到期**：超过 `maxLifetime`
2. **空闲超时**：超过 `idleTimeout`
3. **健康检查失败**：连接已失效
4. **显式驱逐**：调用 `softEvictConnections()`
5. **异常状态**：连接状态异常

### 3. ProxyConnection 创建机制

```java
return poolEntry.createProxyConnection(
    leakTaskFactory.schedule(poolEntry), now);
```

#### 过程
1. **不创建新连接**：`PoolEntry` 里已有真实连接
2. **创建代理对象**：包装成 `ProxyConnection`
3. **注册泄漏检测**：启动定时任务监控连接是否泄漏

#### 为什么需要代理？
- 拦截 `close()` 方法，归还到池中而不是真正关闭
- 跟踪 Statement，确保正确关闭
- 重置连接状态，保证连接可复用

### 4. 泄漏检测机制

#### 工作原理
- 配置 `leakDetectionThreshold`（毫秒）
- 每次借出连接时启动 `ProxyLeakTask`
- 如果在该时间内连接未归还，记录警告日志

#### 如何定位问题？
- 日志包含借出时的调用栈
- 通过堆栈信息定位哪段代码忘记 `close()`

### 5. 构建阶段代码生成

#### 为什么需要？
- `ProxyFactory` 源码中只有占位方法
- 真正的代理类在构建时由 `JavassistProxyFactory` 生成
- 生成专用的字节码代理，性能优于 JDK 动态代理

#### 构建配置

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <executions>
        <execution>
            <phase>compile</phase>
            <goals>
                <goal>exec</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <executable>java</executable>
        <arguments>
            <argument>-cp</argument>
            <argument>...</argument>
            <argument>com.zaxxer.hikari.util.JavassistProxyFactory</argument>
        </arguments>
    </configuration>
</plugin>
```

---

## 常见问题解答

### Q1: 为什么线程本地列表是 List 而不是单个连接？

**A:** 虽然大多数情况下线程只需要一个连接，但：
1. **连接复用缓存**：归还后可能很快又要用
2. **支持嵌套场景**：一个线程可能需要多个连接
3. **限制在50个**：防止无限增长

实际运行中，线程本地列表通常只有 0-2 个连接。

### Q2: requite 怎么读？是什么意思？

**A:** 
- **发音**：/rɪˈkwaɪt/，中文近似音：**瑞-快特**
- **含义**：报答、回报、归还
- **在代码中**：表示将连接归还到池中

### Q3: listener、waiters、sharedList 分别是什么？

**A:**
- **sharedList**：存储所有连接的全局共享列表（`CopyOnWriteArrayList`）
- **waiters**：统计当前等待连接的线程数（`AtomicInteger`）
- **listener**：回调接口，当需要创建新连接时通知 `HikariPool`

### Q4: handoffQueue 是干什么的？

**A:** 
- **作用**：直接交付连接给等待线程
- **类型**：`SynchronousQueue`（无缓冲队列）
- **优势**：避免等待线程被唤醒后还要扫描共享列表

### Q5: 为什么用 CopyOnWriteArrayList？

**A:**
- **读多写少**：连接池中遍历查找是高频操作
- **读操作无锁**：多线程并发读取性能好
- **线程安全**：无需额外同步
- **避免异常**：遍历时不会因并发修改而抛异常

### Q6: 为什么不使用读写锁或 Vector？

**A:**

| 方案 | 问题 |
|------|------|
| **Vector** | 所有操作都加锁，性能差 |
| **ArrayList + synchronized** | 读操作被写操作阻塞 |
| **ReadWriteLock** | 仍有锁开销，写操作会阻塞读操作 |
| **CopyOnWriteArrayList** | ✅ 读操作完全无锁，性能最优 |

---

## 设计决策分析

### CopyOnWriteArrayList vs ReadWriteLock

#### CopyOnWriteArrayList 的优势
1. ✅ **读操作完全无锁**，性能最优
2. ✅ **读操作不会被写操作阻塞**
3. ✅ **实现简单**，无死锁风险
4. ✅ **适合读多写少的场景**

#### ReadWriteLock 的优势
1. ✅ **数据实时性好**（不是快照）
2. ✅ **写操作性能更好**（不复制数组）
3. ✅ **内存占用更少**
4. ✅ **适合读写均衡或写多读少的场景**

#### 选择建议
- **读多写少** → CopyOnWriteArrayList（如连接池）✓
- **读写均衡** → ReadWriteLock
- **写多读少** → ReadWriteLock
- **需要实时数据** → ReadWriteLock
- **需要最高读性能** → CopyOnWriteArrayList

### 为什么选择三层查找机制？

#### 设计优势
1. **性能优化**：优先使用线程本地列表，减少锁竞争
2. **负载均衡**：共享列表允许线程间共享连接
3. **及时响应**：等待队列直接交付，减少唤醒延迟
4. **无锁设计**：大量使用 CAS，避免传统锁的开销
5. **自适应**：根据等待线程数动态创建连接

---

## 总结

HikariCP 的连接池设计体现了以下核心思想：

1. **无锁优先**：尽可能使用无锁设计（CAS、ThreadLocal）
2. **读多写少优化**：针对连接池场景优化（CopyOnWriteArrayList）
3. **三层查找**：线程本地 → 共享列表 → 等待队列
4. **直接交付**：使用 handoffQueue 减少延迟
5. **健康检查**：aliveBypassWindowMs 平衡性能和可靠性

这些设计使得 HikariCP 在保证线程安全的同时，实现了极高的并发性能。

---

## 参考资料

- HikariCP 源码：`src/main/java/com/zaxxer/hikari/`
- 关键类：
  - `HikariPool.java`：连接池主类
  - `ConcurrentBag.java`：连接容器
  - `PoolEntry.java`：连接条目
  - `ProxyConnection.java`：代理连接

