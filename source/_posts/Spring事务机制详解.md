---
title: SpringBoot事务实现原理
author: demus
top: false
cover: false
mathjax: false
toc: true
abbrlink: 104
date: 2023-10-08 10:30:07
categories:
tags:
img:
coverImg:
password:
summary:
keywords:
---
# Spring 事务机制详解

## 目录
1. [Spring @Transactional 实现原理](#1-spring-transactional-实现原理)
2. [Spring AOP 代理机制详解](#2-spring-aop-代理机制详解)
3. [事务传播行为详解](#3-事务传播行为详解)
4. [事务隔离级别](#4-事务隔离级别)
5. [TransactionInterceptor 实现原理](#5-transactioninterceptor-实现原理)

---

## 1. Spring @Transactional 实现原理

### 1.1 核心机制：AOP（面向切面编程）

`@Transactional` 基于 Spring AOP，通过代理对象拦截方法调用，在方法执行前后添加事务逻辑。

### 1.2 实现流程

#### 1.2.1 注解扫描与处理

Spring 在启动时扫描 `@Transactional`，由 `TransactionInterceptor` 处理：

- `@EnableTransactionManagement` 启用事务管理
- `TransactionAttributeSourceAdvisor` 识别带 `@Transactional` 的方法
- 创建代理对象（JDK 动态代理或 CGLIB）

#### 1.2.2 方法拦截执行流程

当调用带 `@Transactional` 的方法时：

```
1. 代理对象拦截方法调用
   ↓
2. TransactionInterceptor.invoke()
   ↓
3. 获取事务属性（传播行为、隔离级别等）
   ↓
4. 获取或创建事务（PlatformTransactionManager）
   ↓
5. 执行业务方法
   ↓
6. 根据执行结果提交或回滚事务
```

### 1.3 关键组件

- **TransactionInterceptor**：核心拦截器，负责方法拦截、事务属性解析、事务管理器调用
- **PlatformTransactionManager**：事务管理器接口
    - `DataSourceTransactionManager`（JDBC）
    - `HibernateTransactionManager`（Hibernate）
    - `JpaTransactionManager`（JPA）
- **TransactionAttributeSource**：解析 `@Transactional` 属性

### 1.4 注意事项

#### 代理失效场景

- 同一个类内部方法调用（绕过代理）
- 方法不是 public
- 异常被捕获未抛出

```java
// ❌ 错误示例：内部调用不会触发事务
public void methodA() {
    this.methodB(); // 不会走代理
}

@Transactional
public void methodB() {
    // 事务不会生效
}
```

```java
// ✅ 正确：通过注入的代理对象调用
@Service
public class UserService {
    
    @Autowired
    private UserService self; // 注入代理对象
    
    public void methodA() {
        self.methodB(); // 会走代理，事务生效
    }
    
    @Transactional
    public void methodB() {
        // 事务生效
    }
}
```

---

## 2. Spring AOP 代理机制详解

### 2.1 Spring AOP 与 AspectJ 的关系

#### 2.1.1 核心区别

**Spring AOP 和 AspectJ 不是同一级别的**，它们是两种不同的 AOP 实现方式：

```
AOP 实现方式（顶层）
    ├── Spring AOP（一种实现方式）
    │   └── 使用代理模式
    │       ├── JDK 动态代理
    │       └── CGLIB 代理
    │
    └── AspectJ（另一种实现方式）
        └── 使用字节码织入（不是代理）
            ├── 编译时织入
            ├── 编译后织入
            └── 加载时织入（LTW）
```

#### 2.1.2 Spring AOP：运行时动态代理

**实现方式**：
- **JDK 动态代理**：基于接口，使用 `java.lang.reflect.Proxy`
- **CGLIB 代理**：基于子类继承，运行时生成字节码

**特点**：
- 在运行时生成代理对象
- 仅支持方法级别的拦截
- 仅支持 Spring Bean

#### 2.1.3 AspectJ：编译时/加载时织入（不是动态代理）

**织入时机**：
- **编译时织入**：编译时修改字节码
- **编译后织入**：对已编译的 .class 文件进行织入
- **加载时织入**：类加载时通过 ClassLoader 织入

**特点**：
- 不是动态代理，而是字节码织入
- 支持字段、构造器、静态初始化等
- 支持任意 Java 类

#### 2.1.4 对比总结

| 特性 | Spring AOP | AspectJ |
|------|-----------|---------|
| 实现方式 | 动态代理（运行时） | 字节码织入 |
| 织入时机 | 运行时 | 编译时/编译后/加载时 |
| 性能 | 相对较慢（代理调用） | 更快（直接执行） |
| 功能范围 | 仅方法级别 | 字段、构造器、静态初始化等 |
| 依赖 | 仅 Spring 容器 | 需要 AspectJ 编译器/Weaver |
| 限制 | 仅 Spring Bean | 任意 Java 类 |

#### 2.1.5 Spring 中使用 AspectJ 的方式

**方式一：Spring AOP（默认，运行时代理）**
```java
// 使用 AspectJ 注解风格，但底层是 Spring AOP
@Aspect
@Component
public class MyAspect {
    @Before("execution(* com.example.*.*(..))")
    public void before() {
        // ...
    }
}
```
- 需要 `@EnableAspectJAutoProxy`
- 运行时动态代理
- 仅支持 Spring Bean

**方式二：AspectJ LTW（加载时织入）**
```java
// 需要配置 aop.xml
@Configuration
@EnableLoadTimeWeaving
public class AspectJConfig {
    // ...
}
```
- 需要 `aspectjweaver.jar`
- 类加载时织入
- 支持任意 Java 类

### 2.2 CGLIB 与 AspectJ 在 Spring 中的配合

#### 2.2.1 核心关系总结

- **CGLIB 和 AspectJ 是两种不同的技术**，用于不同的目的
- **Spring AOP 使用 CGLIB 作为代理实现方式之一**
- **AspectJ 提供切点表达式解析**，Spring AOP 借用其注解风格

#### 2.2.2 CGLIB 是什么

CGLIB（Code Generation Library）是一个字节码生成库，用于在运行时生成类的子类。

**特点**：
- 运行时生成代理类（字节码生成）
- 基于继承（创建目标类的子类）
- 不需要接口
- 性能较好（比 JDK 动态代理快）

**工作原理**：
```java
// 原始类
public class UserService {
    public void save() {
        // ...
    }
}

// CGLIB 生成的代理类（运行时生成）
public class UserService$$EnhancerByCGLIB$$xxx extends UserService {
    private MethodInterceptor interceptor;
    
    @Override
    public void save() {
        // 调用拦截器
        interceptor.intercept(this, method, args, methodProxy);
    }
}
```

#### 2.2.3 AspectJ 在 Spring 中的作用

AspectJ 在 Spring AOP 中提供：
- 切点表达式语言（`execution`, `within` 等）
- 注解支持（`@Aspect`, `@Before`, `@After` 等）
- 表达式解析器（`aspectjweaver.jar`）

#### 2.2.4 在 Spring 中的完整关系

```
Spring AOP
    ├── 使用 AspectJ 的注解风格 (@Aspect, @Before, @After...)
    ├── 使用 AspectJ 的切点表达式 (execution, within...)
    ├── 使用 AspectJ 的解析器 (aspectjweaver.jar)
    └── 使用两种代理实现：
        ├── JDK 动态代理 (基于接口)
        └── CGLIB 代理 (基于继承) ← 这里用到 CGLIB
```

**简单记忆**：
- **AspectJ**：提供语法和解析器（告诉 Spring 切哪里）
- **CGLIB**：生成代理类（告诉 Spring 怎么切）

两者在 Spring AOP 中配合使用，但职责不同。

### 2.3 代理类的生成时机

#### 2.3.1 核心答案

**代理类在 Spring 启动时（Bean 创建时）生成，不是在实际调用时生成**

#### 2.3.2 详细流程

**阶段1：Spring 启动时（Bean 创建阶段）**

```
1. 扫描所有 @Component、@Service 等注解的类
   ↓
2. 识别带 @Transactional 的类和方法
   ↓
3. 创建 Bean 定义（BeanDefinition）
   ↓
4. 检查是否需要代理
   ↓
5. 创建代理对象（此时就生成了！）
   ↓
6. 将代理对象放入容器
```

**阶段2：实际调用时（运行时）**

```
1. 从容器获取 Bean（已经是代理对象了！）
   ↓
2. 调用方法
   ↓
3. 代理对象拦截调用
   ↓
4. TransactionInterceptor 处理
```

#### 2.3.3 完整时序图

```
Spring 启动阶段（Bean 创建时）
│
├─ 1. 扫描所有 Bean
│   └─ 发现 @Service class UserService
│
├─ 2. 检查是否需要代理
│   └─ 发现 @Transactional 注解
│
├─ 3. 创建代理对象（此时生成！）
│   ├─ 使用 CGLIB 或 JDK 动态代理
│   ├─ 生成：UserService$$EnhancerBySpringCGLIB$$xxx
│   └─ 添加 TransactionInterceptor
│
└─ 4. 将代理对象放入容器
    └─ 容器中存储的是代理对象，不是原始对象

─────────────────────────────────────────

运行时（实际调用时）
│
├─ 1. 从容器获取 Bean
│   └─ 返回的是代理对象（已存在）
│
├─ 2. 调用方法
│   └─ userService.saveUser(user)
│
├─ 3. 代理对象拦截
│   └─ UserService$$EnhancerBySpringCGLIB$$xxx.saveUser()
│
├─ 4. TransactionInterceptor.invoke()
│   ├─ 获取事务属性
│   ├─ 创建/获取事务
│   ├─ 调用原始方法
│   └─ 提交/回滚事务
│
└─ 5. 返回结果
```

#### 2.3.4 代理类命名规则

**JDK 动态代理**：
```java
$Proxy0
$Proxy1
$Proxy2
...
```

**CGLIB 代理**：
```java
// CGLIB 代理生成的类名格式
原始类名$$EnhancerByCGLIB$$hashcode
```

**Spring AOP 使用 CGLIB**：
```java
// Spring AOP 使用 CGLIB 时的类名格式
原始类名$$EnhancerBySpringCGLIB$$hashcode
```

#### 2.3.5 关键点总结

- **代理对象生成时机**：Spring 启动时（Bean 创建阶段）
- **不是在调用时生成**：调用时只是使用已存在的代理对象
- **为什么在启动时生成**：
    - 性能：避免每次调用都创建代理
    - 一致性：同一个 Bean 始终使用同一个代理对象
    - 容器管理：代理对象作为 Bean 被 Spring 管理

---

## 3. 事务传播行为详解

### 3.1 最常用的三个传播行为

#### 1. REQUIRED（默认，最常用）

**含义**：
- 如果当前存在事务，则加入该事务
- 如果当前不存在事务，则创建一个新事务

**使用场景**：
- 大多数业务方法
- 默认行为，无需显式指定

**代码示例**：
```java
@Service
public class UserService {
    
    @Autowired
    private OrderService orderService;
    
    // 默认就是 REQUIRED，可以不写
    @Transactional  // 等同于 @Transactional(propagation = Propagation.REQUIRED)
    public void createUserAndOrder(User user) {
        // 创建用户
        userMapper.insert(user);
        
        // 调用另一个事务方法
        orderService.createOrder(user.getId());  // 会加入当前事务
    }
}

@Service
public class OrderService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder(Long userId) {
        // 如果从 createUserAndOrder 调用，会加入外层事务
        // 如果单独调用，会创建新事务
        orderMapper.insert(new Order(userId));
    }
}
```

#### 2. REQUIRES_NEW（第二常用）

**含义**：
- 总是创建一个新事务
- 如果当前存在事务，则挂起当前事务，创建新事务
- 新事务独立提交或回滚

**使用场景**：
- 日志记录（即使主业务失败也要记录）
- 审计操作
- 需要独立事务的业务

**代码示例**：
```java
@Service
public class OrderService {
    
    @Autowired
    private LogService logService;
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder(Order order) {
        try {
            // 主业务逻辑
            orderMapper.insert(order);
            
            // 记录日志（需要独立事务）
            logService.saveLog("订单创建成功", order.getId());
            
        } catch (Exception e) {
            // 即使这里回滚，日志也会保存
            throw e;
        }
    }
}

@Service
public class LogService {
    
    // 使用 REQUIRES_NEW，确保日志总是被保存
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(String message, Long orderId) {
        // 这个操作会在新事务中执行
        // 即使外层事务回滚，这个也会提交
        logMapper.insert(new Log(message, orderId));
    }
}
```

#### 3. NESTED（第三常用）

**含义**：
- 如果当前存在事务，则创建一个嵌套事务（保存点）
- 如果当前不存在事务，则创建一个新事务
- 嵌套事务可以独立回滚，不影响外层事务

**使用场景**：
- 批量操作中的单个失败处理
- 需要部分回滚的场景
- 子操作失败不影响主操作

**代码示例**：
```java
@Service
public class BatchService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void batchProcessOrders(List<Order> orders) {
        for (Order order : orders) {
            try {
                // 每个订单处理是嵌套事务
                processSingleOrder(order);
            } catch (Exception e) {
                // 单个订单失败不影响其他订单
                log.error("订单处理失败: " + order.getId(), e);
                // 嵌套事务回滚，外层事务继续
            }
        }
    }
    
    // 使用 NESTED，创建嵌套事务（保存点）
    @Transactional(propagation = Propagation.NESTED)
    public void processSingleOrder(Order order) {
        // 验证订单
        validateOrder(order);
        
        // 扣减库存
        reduceStock(order);
        
        // 创建订单
        orderMapper.insert(order);
        
        // 如果这里抛异常，只回滚这个嵌套事务
        // 不影响 batchProcessOrders 中的其他订单
    }
}
```

### 3.2 三个传播行为对比

| 传播行为 | 当前有事务 | 当前无事务 | 回滚影响 | 使用频率 |
|---------|-----------|-----------|---------|---------|
| **REQUIRED** | 加入事务 | 创建新事务 | 一起回滚 | ⭐⭐⭐⭐⭐ |
| **REQUIRES_NEW** | 挂起，创建新事务 | 创建新事务 | 独立回滚 | ⭐⭐⭐⭐ |
| **NESTED** | 创建嵌套事务 | 创建新事务 | 可独立回滚 | ⭐⭐⭐ |

### 3.3 REQUIRES_NEW 和 NESTED 的区别

#### REQUIRES_NEW：互不影响（完全独立）

- 创建完全独立的新事务
- 外层事务回滚 → 不影响内层（内层已独立提交）
- 内层事务回滚 → 不影响外层（外层继续执行）

#### NESTED：有影响（单向）

- 创建嵌套事务（保存点）
- 外层回滚 → 影响内层（一起回滚）
- 内层回滚 → 不影响外层（回滚到保存点）

**对比表格**：

| 特性 | REQUIRES_NEW | NESTED |
|------|-------------|--------|
| 事务关系 | 完全独立 | 嵌套（保存点） |
| 外层回滚影响内层 | ❌ 不影响（内层已提交） | ✅ 影响（一起回滚） |
| 内层回滚影响外层 | ❌ 不影响（外层继续） | ❌ 不影响（回滚到保存点） |
| 提交时机 | 内层立即提交 | 外层提交时一起提交 |
| 使用场景 | 日志、审计（必须保存） | 批量操作（部分回滚） |

---

## 4. 事务隔离级别

### 4.1 可以设置不同的隔离级别

每个 `@Transactional` 注解都可以独立配置自己的隔离级别，互不影响。

### 4.2 如何设置不同的隔离级别

```java
@Service
public class OrderService {
    
    // 方法1：使用默认隔离级别（通常是 READ_COMMITTED）
    @Transactional
    public void createOrder(Order order) {
        orderMapper.insert(order);
    }
    
    // 方法2：使用 READ_UNCOMMITTED（最低隔离级别，性能最好）
    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    public void quickQuery(Long orderId) {
        return orderMapper.selectById(orderId);
    }
    
    // 方法3：使用 READ_COMMITTED（默认，大多数场景）
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void updateOrder(Order order) {
        orderMapper.updateById(order);
    }
    
    // 方法4：使用 REPEATABLE_READ（可重复读）
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void calculateTotalAmount(Long userId) {
        // 需要多次读取一致性的场景
        List<Order> orders = orderMapper.selectByUserId(userId);
        BigDecimal total = orders.stream()
            .map(Order::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return total;
    }
    
    // 方法5：使用 SERIALIZABLE（最高隔离级别，最严格）
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void criticalBalanceUpdate(Long accountId, BigDecimal amount) {
        // 关键业务，需要最高级别的隔离
        Account account = accountMapper.selectById(accountId);
        account.setBalance(account.getBalance().add(amount));
        accountMapper.updateById(account);
    }
}
```

### 4.3 五种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 | 使用场景 |
|---------|------|-----------|------|------|---------|
| **READ_UNCOMMITTED** | ❌ 可能 | ❌ 可能 | ❌ 可能 | ⭐⭐⭐⭐⭐ | 统计查询、性能优先 |
| **READ_COMMITTED** | ✅ 避免 | ❌ 可能 | ❌ 可能 | ⭐⭐⭐⭐ | 大多数业务（默认） |
| **REPEATABLE_READ** | ✅ 避免 | ✅ 避免 | ❌ 可能 | ⭐⭐⭐ | 需要多次读取一致 |
| **SERIALIZABLE** | ✅ 避免 | ✅ 避免 | ✅ 避免 | ⭐ | 关键业务、金融操作 |

### 4.4 隔离级别的实现原理

虽然数据库有全局隔离级别，但 Spring 可以在每个事务连接上设置不同的隔离级别：

```
Spring 事务管理器的工作流程
1. 开启事务时，从连接池获取 Connection
   ↓
2. 为这个 Connection 设置隔离级别
   Connection.setTransactionIsolation(Isolation.REPEATABLE_READ)
   ↓
3. 执行业务方法
   ↓
4. 提交/回滚事务
   ↓
5. 归还 Connection 到连接池
   （隔离级别会重置为连接池的默认值）
```

**关键点**：
- 每个事务使用独立的数据库连接
- Spring 在事务开始时为连接设置隔离级别
- 通过 `Connection.setTransactionIsolation()` 实现
- 事务结束后连接归还连接池

### 4.5 注意事项

1. **隔离级别不能降级**：内层事务不能使用比外层更低的隔离级别
2. **数据库支持情况**：不同数据库支持的隔离级别不同
3. **性能考虑**：根据业务需求选择合适的隔离级别

---

## 5. TransactionInterceptor 实现原理

### 5.1 TransactionInterceptor 入口

```java
// Spring 源码：TransactionInterceptor
public class TransactionInterceptor implements MethodInterceptor {
    
    private PlatformTransactionManager transactionManager;
    private TransactionAttributeSource transactionAttributeSource;
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. 获取目标类和方法
        Class<?> targetClass = invocation.getThis() != null ? 
            AopUtils.getTargetClass(invocation.getThis()) : null;
        Method method = invocation.getMethod();
        
        // 2. 获取事务属性（从 @Transactional 注解解析）
        TransactionAttribute txAttr = 
            transactionAttributeSource.getTransactionAttribute(method, targetClass);
        
        // 3. 获取事务管理器
        PlatformTransactionManager tm = determineTransactionManager(txAttr);
        
        // 4. 创建事务信息（关键：这里处理传播行为）
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, method, targetClass);
        
        Object retVal;
        try {
            // 5. 执行业务方法
            retVal = invocation.proceed();
            
            // 6. 提交事务
            commitTransactionAfterReturning(txInfo);
            return retVal;
            
        } catch (Throwable ex) {
            // 7. 回滚事务
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // 8. 清理事务信息
            cleanupTransactionInfo(txInfo);
        }
    }
}
```

### 5.2 PlatformTransactionManager 获取事务（处理传播行为）

```java
// Spring 源码：AbstractPlatformTransactionManager
public abstract class AbstractPlatformTransactionManager 
        implements PlatformTransactionManager {
    
    @Override
    public final TransactionStatus getTransaction(TransactionDefinition definition) 
            throws TransactionException {
        
        // 1. 获取当前事务（可能为 null）
        Object transaction = doGetTransaction();
        
        // 2. 检查当前是否存在事务
        if (isExistingTransaction(transaction)) {
            // 处理已存在事务的情况（根据传播行为）
            return handleExistingTransaction(definition, transaction, debugEnabled);
        }
        
        // 3. 当前不存在事务，需要创建新事务
        if (definition.getPropagationBehavior() == 
                TransactionDefinition.PROPAGATION_REQUIRED ||
               definition.getPropagationBehavior() == 
                TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
               definition.getPropagationBehavior() == 
                TransactionDefinition.PROPAGATION_NESTED) {
            // 挂起当前事务（如果有）
            SuspendedResourcesHolder suspendedResources = suspend(null);
            try {
                // 创建新事务
                return startTransaction(definition, transaction, debugEnabled, 
                    suspendedResources);
            } catch (RuntimeException | Error ex) {
                resume(null, suspendedResources);
                throw ex;
            }
        }
    }
    
    // 处理已存在事务的情况（关键方法）
    private TransactionStatus handleExistingTransaction(
            TransactionDefinition definition, 
            Object transaction, 
            boolean debugEnabled) throws TransactionException {
        
        // PROPAGATION_REQUIRES_NEW：挂起当前事务，创建新事务
        if (definition.getPropagationBehavior() == 
                TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
            // 挂起当前事务
            SuspendedResourcesHolder suspendedResources = suspend(transaction);
            try {
                // 创建新事务
                return startTransaction(definition, transaction, debugEnabled, 
                    suspendedResources);
            } catch (RuntimeException | Error beginEx) {
                resumeAfterBeginException(transaction, suspendedResources, beginEx);
                throw beginEx;
            }
        }
        
        // PROPAGATION_NESTED：创建嵌套事务（保存点）
        if (definition.getPropagationBehavior() == 
                TransactionDefinition.PROPAGATION_NESTED) {
            if (!isNestedTransactionAllowed()) {
                throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions");
            }
            // 创建保存点
            if (useSavepointForNestedTransaction()) {
                DefaultTransactionStatus status = prepareTransactionStatus(
                    definition, transaction, false, debugEnabled, null, null);
                status.createAndHoldSavepoint();
                return status;
            } else {
                return startTransaction(definition, transaction, debugEnabled, null);
            }
        }
        
        // PROPAGATION_REQUIRED：加入当前事务
        // 验证隔离级别（不能降级）
        validateExistingTransaction(definition);
        // 加入当前事务，不创建新事务
        return prepareTransactionStatus(definition, transaction, false, 
            debugEnabled, null, newSynchronization);
    }
}
```

### 5.3 具体传播行为的实现

#### 5.3.1 REQUIRED（默认）的实现

```java
// REQUIRED 传播行为的处理逻辑
private TransactionStatus handleRequired(
        TransactionDefinition definition, 
        Object transaction) {
    
    // 如果当前存在事务
    if (isExistingTransaction(transaction)) {
        // 加入当前事务（不创建新事务）
        validateExistingTransaction(definition);
        return prepareTransactionStatus(definition, transaction, false, 
            debugEnabled, null, newSynchronization);
    }
    
    // 如果当前不存在事务，创建新事务
    return startTransaction(definition, transaction, debugEnabled, null);
}
```

#### 5.3.2 REQUIRES_NEW 的实现

```java
// REQUIRES_NEW 传播行为的处理逻辑
private TransactionStatus handleRequiresNew(
        TransactionDefinition definition, 
        Object transaction) {
    
    // 如果当前存在事务
    if (isExistingTransaction(transaction)) {
        // 1. 挂起当前事务
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            // 2. 创建新事务（独立事务）
            return startTransaction(definition, transaction, debugEnabled, 
                suspendedResources);
        } catch (RuntimeException | Error beginEx) {
            // 如果创建新事务失败，恢复被挂起的事务
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }
    
    // 如果当前不存在事务，直接创建新事务
    return startTransaction(definition, transaction, debugEnabled, null);
}

// 挂起当前事务
protected final SuspendedResourcesHolder suspend(Object transaction) {
    // 保存当前事务的所有信息
    // 清理当前线程的事务信息
    // 返回被挂起的事务信息
}

// 恢复被挂起的事务
protected final void resume(Object transaction, 
        SuspendedResourcesHolder resourcesHolder) {
    // 恢复被挂起的事务信息
    // 重新设置到当前线程
}
```

#### 5.3.3 NESTED 的实现

```java
// NESTED 传播行为的处理逻辑
private TransactionStatus handleNested(
        TransactionDefinition definition, 
        Object transaction) {
    
    // 检查是否支持嵌套事务
    if (!isNestedTransactionAllowed()) {
        throw new NestedTransactionNotSupportedException(
            "Transaction manager does not allow nested transactions");
    }
    
    // 如果当前存在事务
    if (isExistingTransaction(transaction)) {
        // 使用保存点（Savepoint）实现嵌套事务
        if (useSavepointForNestedTransaction()) {
            // 创建保存点
            DefaultTransactionStatus status = prepareTransactionStatus(
                definition, transaction, false, debugEnabled, null, null);
            // 创建并持有保存点
            status.createAndHoldSavepoint();
            return status;
        } else {
            // 某些事务管理器不支持保存点，创建新事务
            return startTransaction(definition, transaction, debugEnabled, null);
        }
    }
    
    // 如果当前不存在事务，创建新事务
    return startTransaction(definition, transaction, debugEnabled, null);
}

// 创建保存点（DataSourceTransactionManager 实现）
protected void doSetSavepoint(Object savepoint) throws TransactionException {
    ConnectionHolder conHolder = (ConnectionHolder) 
        TransactionSynchronizationManager.getResource(obtainDataSource());
    try {
        // 在数据库连接上创建保存点
        Savepoint savepointToUse = conHolder.getConnection()
            .setSavepoint(savepointName);
        conHolder.setSavepoint(savepointToUse);
    } catch (SQLException ex) {
        throw new CannotCreateTransactionException(
            "Could not create JDBC savepoint", ex);
    }
}

// 回滚到保存点
protected void doRollbackToSavepoint(Object savepoint) throws TransactionException {
    ConnectionHolder conHolder = (ConnectionHolder) 
        TransactionSynchronizationManager.getResource(obtainDataSource());
    try {
        // 回滚到保存点
        conHolder.getConnection().rollback((Savepoint) savepoint);
    } catch (SQLException ex) {
        throw new TransactionSystemException(
            "Could not roll back to JDBC savepoint", ex);
    }
}
```

### 5.4 事务提交和回滚的处理

```java
// 提交事务
protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
    if (txInfo != null && txInfo.hasTransaction()) {
        // 提交事务
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}

// AbstractPlatformTransactionManager 的提交逻辑
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    
    // 如果是嵌套事务（有保存点）
    if (defStatus.hasSavepoint()) {
        // 只释放保存点，不提交外层事务
        status.releaseHeldSavepoint();
    } 
    // 如果是新事务（REQUIRES_NEW 或独立事务）
    else if (status.isNewTransaction()) {
        // 提交事务
        doCommit(status);
    } 
    // 如果是加入的事务（REQUIRED），不提交，等待外层提交
    else {
        // 不提交，外层事务提交时会一起提交
    }
}
```

### 5.5 完整执行流程示例

```java
@Service
public class OrderService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void method1() {
        // 1. TransactionInterceptor.invoke()
        // 2. createTransactionIfNecessary()
        // 3. tm.getTransaction() -> handleExistingTransaction()
        //    -> 当前无事务，创建新事务
        // 4. 执行业务代码
        method2(); // 调用 REQUIRES_NEW
        method3(); // 调用 NESTED
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void method2() {
        // 1. TransactionInterceptor.invoke()
        // 2. createTransactionIfNecessary()
        // 3. tm.getTransaction() -> handleExistingTransaction()
        //    -> 检测到当前有事务
        //    -> suspend(transaction) 挂起外层事务
        //    -> startTransaction() 创建新事务
        // 4. 执行业务代码
        // 5. 提交新事务
        // 6. resume() 恢复外层事务
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void method3() {
        // 1. TransactionInterceptor.invoke()
        // 2. createTransactionIfNecessary()
        // 3. tm.getTransaction() -> handleExistingTransaction()
        //    -> 检测到当前有事务
        //    -> createAndHoldSavepoint() 创建保存点
        // 4. 执行业务代码
        // 5. 如果成功：releaseHeldSavepoint() 释放保存点
        //    如果失败：rollbackToSavepoint() 回滚到保存点
    }
}
```

### 5.6 关键实现点总结

1. **REQUIRED（默认）**
    - 有事务：加入当前事务
    - 无事务：创建新事务

2. **REQUIRES_NEW**
    - 有事务：挂起当前事务 → 创建新事务 → 提交新事务 → 恢复外层事务
    - 无事务：创建新事务

3. **NESTED**
    - 有事务：创建保存点（Savepoint）→ 失败回滚到保存点，成功释放保存点
    - 无事务：创建新事务

### 5.7 核心方法

- `handleExistingTransaction()`：处理已存在事务的情况
- `suspend()`：挂起事务
- `resume()`：恢复事务
- `createAndHoldSavepoint()`：创建保存点
- `processCommit()`：处理提交（区分新事务、加入事务、嵌套事务）

---

## 总结

### 关键要点

1. **Spring @Transactional 基于 AOP 实现**，通过代理对象拦截方法调用
2. **代理对象在 Spring 启动时生成**，不是在实际调用时生成
3. **Spring AOP 使用 AspectJ 的注解和表达式**，但底层是动态代理（JDK 或 CGLIB）
4. **CGLIB 是代理实现方式**，AspectJ 提供语法和解析器
5. **事务传播行为通过 PlatformTransactionManager 实现**，不同传播行为有不同的处理逻辑
6. **每个事务可以设置不同的隔离级别**，通过 Connection.setTransactionIsolation() 实现

### 学习建议

- 理解 Spring AOP 和 AspectJ 的区别
- 掌握三种常用传播行为的使用场景
- 了解代理对象的生成时机
- 理解事务传播机制的底层实现原理

---

*本文档整理自 Spring 事务机制相关技术讨论*

