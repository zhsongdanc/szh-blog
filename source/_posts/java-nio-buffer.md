---
title: Java NIO Buffer 使用与实现原理
author: demus
top: false
cover: false
mathjax: false
toc: true
abbrlink: 128001
date: 2023-10-08 10:30:07
categories:
tags:
img:
coverImg:
password:
summary:
keywords:
---

这是第一篇博客

# Java NIO Buffer 使用与实现原理

## 1. Buffer 概述

### 1.1 什么是 Buffer

Buffer（缓冲区）是 Java NIO 中用于与通道（Channel）进行数据交互的对象。它是一个线性的、有限的数据容器，本质上是一个数组，但提供了更丰富的操作接口。

### 1.2 Buffer 的核心特性

- **容量（Capacity）**：Buffer 的最大数据容量，创建后不可改变
- **位置（Position）**：下一个要读取或写入的索引位置
- **限制（Limit）**：第一个不应该读取或写入的索引位置
- **标记（Mark）**：一个备忘位置，可以通过 `reset()` 恢复到该位置

### 1.3 Buffer 的类型

Java NIO 提供了以下类型的 Buffer：

- `ByteBuffer`
- `CharBuffer`
- `ShortBuffer`
- `IntBuffer`
- `LongBuffer`
- `FloatBuffer`
- `DoubleBuffer`

其中 `ByteBuffer` 是最常用的，其他类型都是基于 `ByteBuffer` 的视图。

## 2. Buffer 的基本使用

### 2.1 创建 Buffer

```java
// 方式1：分配指定容量的 Buffer（堆内存）
ByteBuffer buffer = ByteBuffer.allocate(1024);

// 方式2：分配直接内存 Buffer（堆外内存）
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);

// 方式3：包装现有数组
byte[] array = new byte[1024];
ByteBuffer wrappedBuffer = ByteBuffer.wrap(array);
```

### 2.2 写入数据到 Buffer

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

// 方式1：使用 put() 方法
buffer.put((byte) 'H');
buffer.put((byte) 'e');
buffer.put((byte) 'l');
buffer.put((byte) 'l');
buffer.put((byte) 'o');

// 方式2：批量写入
byte[] data = "Hello".getBytes();
buffer.put(data);

// 方式3：从 Channel 读取数据到 Buffer
channel.read(buffer);
```

### 2.3 从 Buffer 读取数据

```java
// 切换为读模式
buffer.flip();

// 方式1：逐个读取
while (buffer.hasRemaining()) {
    byte b = buffer.get();
    System.out.print((char) b);
}

// 方式2：批量读取
byte[] dest = new byte[buffer.remaining()];
buffer.get(dest);

// 方式3：写入到 Channel
channel.write(buffer);
```

### 2.4 Buffer 的状态转换

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

// 初始状态：position=0, limit=capacity
System.out.println("初始: position=" + buffer.position() + ", limit=" + buffer.limit());

// 写入数据后：position 移动到写入位置
buffer.put("Hello".getBytes());
System.out.println("写入后: position=" + buffer.position() + ", limit=" + buffer.limit());

// flip()：切换为读模式，limit=position, position=0
buffer.flip();
System.out.println("flip后: position=" + buffer.position() + ", limit=" + buffer.limit());

// 读取数据后：position 移动到读取位置
byte[] dest = new byte[3];
buffer.get(dest);
System.out.println("读取后: position=" + buffer.position() + ", limit=" + buffer.limit());

// rewind()：重新读取，position=0, limit 不变
buffer.rewind();
System.out.println("rewind后: position=" + buffer.position() + ", limit=" + buffer.limit());

// clear()：清空 Buffer，position=0, limit=capacity（但数据未清除）
buffer.clear();
System.out.println("clear后: position=" + buffer.position() + ", limit=" + buffer.limit());
```

## 3. Buffer 核心方法详解

### 3.1 flip() - 切换为读模式

```java
public final Buffer flip() {
    limit = position;  // 将 limit 设置为当前 position
    position = 0;      // 将 position 重置为 0
    mark = -1;         // 清除标记
    return this;
}
```

**使用场景**：写入数据后，准备读取数据时调用。

### 3.2 clear() - 清空 Buffer

```java
public final Buffer clear() {
    position = 0;      // position 重置为 0
    limit = capacity;  // limit 设置为 capacity
    mark = -1;         // 清除标记
    return this;
}
```

**注意**：`clear()` 不会清除 Buffer 中的数据，只是重置了位置指针，数据仍然存在。

### 3.3 rewind() - 重新读取

```java
public final Buffer rewind() {
    position = 0;      // position 重置为 0
    mark = -1;         // 清除标记
    return this;
}
```

**使用场景**：需要重新读取 Buffer 中的数据，但 limit 保持不变。

### 3.4 compact() - 压缩 Buffer

```java
public ByteBuffer compact() {
    System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
    position(remaining());
    limit(capacity());
    discardMark();
    return this;
}
```

**作用**：将未读取的数据移动到 Buffer 的开头，为后续写入腾出空间。

**使用场景**：部分读取数据后，需要继续写入新数据。

### 3.5 mark() 和 reset() - 标记和重置

```java
public final Buffer mark() {
    mark = position;  // 记录当前 position
    return this;
}

public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;  // 恢复到标记位置
    return this;
}
```

## 4. Buffer 实现原理深度剖析

### 4.1 Buffer 的类层次结构

```
Buffer (抽象类)
  ├── ByteBuffer (抽象类)
  │   ├── HeapByteBuffer (堆内存实现)
  │   └── DirectByteBuffer (直接内存实现)
  ├── CharBuffer
  ├── ShortBuffer
  ├── IntBuffer
  ├── LongBuffer
  ├── FloatBuffer
  └── DoubleBuffer
```

### 4.2 Buffer 核心字段

```java
public abstract class Buffer {
    // 标记位置
    private int mark = -1;
    
    // 当前位置
    private int position = 0;
    
    // 限制位置
    private int limit;
    
    // 容量
    private int capacity;
    
    // 地址（用于直接内存）
    long address;
}
```

### 4.3 HeapByteBuffer 实现

#### 4.3.1 内部结构

```java
class HeapByteBuffer extends ByteBuffer {
    // 底层数组
    protected final byte[] hb;
    
    // 数组偏移量
    protected final int offset;
    
    // 是否为只读
    boolean isReadOnly;
    
    HeapByteBuffer(int cap, int lim) {
        super(-1, 0, lim, cap, new byte[cap], 0);
        // hb 在父类中初始化
    }
}
```

#### 4.3.2 数据访问实现

```java
// 获取数据
public byte get() {
    return hb[ix(nextGetIndex())];
}

public byte get(int i) {
    return hb[ix(checkIndex(i))];
}

// 写入数据
public ByteBuffer put(byte x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}

public ByteBuffer put(int i, byte x) {
    hb[ix(checkIndex(i))] = x;
    return this;
}

// 计算实际索引
protected int ix(int i) {
    return i + offset;
}
```

**关键点**：
- `HeapByteBuffer` 使用 Java 堆内存中的 `byte[]` 数组
- 所有操作都是对数组的直接访问
- 受 GC 管理，内存分配和回收由 JVM 控制

### 4.4 DirectByteBuffer 实现

#### 4.4.1 内部结构

```java
class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer {
    // 直接内存地址
    protected long address;
    
    // 内存分配器
    private final Cleaner cleaner;
    
    DirectByteBuffer(int cap) {
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);
        
        long base = 0;
        try {
            // 分配直接内存
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        // 创建清理器，用于释放直接内存
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
}
```

#### 4.4.2 内存分配机制

```java
// 使用 Unsafe 分配直接内存
long base = unsafe.allocateMemory(size);

// 设置内存内容
unsafe.setMemory(base, size, (byte) 0);

// 计算对齐后的地址
if (pa && (base % ps != 0)) {
    address = base + ps - (base & (ps - 1));
} else {
    address = base;
}
```

**关键点**：
- `DirectByteBuffer` 使用堆外内存（直接内存）
- 通过 `Unsafe.allocateMemory()` 分配
- 不受 GC 直接管理，需要手动释放
- 使用 `Cleaner` 机制在对象被 GC 时自动释放内存

#### 4.4.3 内存释放机制

```java
private static class Deallocator implements Runnable {
    private static Unsafe unsafe = Unsafe.getUnsafe();
    
    private long address;
    private long size;
    private int capacity;
    
    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }
    
    public void run() {
        if (address == 0) {
            return;
        }
        // 释放直接内存
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}
```

**内存释放流程**：
1. `DirectByteBuffer` 对象被 GC 回收
2. `Cleaner` 检测到对象被回收
3. 调用 `Deallocator.run()` 释放直接内存
4. 调用 `Unsafe.freeMemory()` 释放内存

### 4.5 Buffer 的视图机制

#### 4.5.1 创建视图

```java
// 创建只读视图
public ByteBuffer asReadOnlyBuffer() {
    return new HeapByteBufferR(hb, this.markValue(), this.position(), 
                               this.limit(), this.capacity(), offset);
}

// 创建其他类型的视图
public CharBuffer asCharBuffer() {
    int size = this.remaining() >> 1;
    int off = offset + position();
    return (bigEndian
            ? (CharBuffer)(new ByteBufferAsCharBufferB(this, -1, 0, size, size, off))
            : (CharBuffer)(new ByteBufferAsCharBufferL(this, -1, 0, size, size, off)));
}
```

#### 4.5.2 视图共享机制

```java
// 视图 Buffer 共享底层数组
class HeapByteBufferR extends HeapByteBuffer {
    HeapByteBufferR(byte[] buf, int mark, int pos, int lim, int cap, int off) {
        super(buf, mark, pos, lim, cap, off);
        this.isReadOnly = true;  // 标记为只读
    }
    
    // 所有修改操作都抛出异常
    public ByteBuffer put(byte x) {
        throw new ReadOnlyBufferException();
    }
}
```

**关键点**：
- 视图 Buffer 共享底层数据数组
- 修改一个 Buffer 会影响其他视图
- 但每个视图有独立的 position、limit、mark

### 4.6 Buffer 的字节序（ByteOrder）

```java
public abstract class ByteBuffer extends Buffer implements Comparable<ByteBuffer> {
    // 字节序：大端（BIG_ENDIAN）或小端（LITTLE_ENDIAN）
    final boolean bigEndian;
    
    // 默认使用大端序
    ByteBuffer(int mark, int pos, int lim, int cap, byte[] hb, int offset) {
        super(mark, pos, lim, cap);
        this.hb = hb;
        this.offset = offset;
        this.isReadOnly = false;
        this.bigEndian = true;  // 默认大端序
    }
    
    // 设置字节序
    public final ByteBuffer order(ByteOrder bo) {
        bigEndian = (bo == ByteOrder.BIG_ENDIAN);
        nativeByteOrder = false;
        return this;
    }
}
```

**字节序影响**：
- 多字节数据类型（int、long 等）的存储顺序
- 大端序：高位字节在前（网络字节序）
- 小端序：低位字节在前（x86 架构默认）

## 5. Buffer 使用最佳实践

### 5.1 堆内存 vs 直接内存

| 特性 | HeapByteBuffer | DirectByteBuffer |
|------|---------------|------------------|
| 内存位置 | 堆内存 | 堆外内存 |
| 分配速度 | 快 | 慢 |
| 访问速度 | 相对慢 | 相对快 |
| GC 管理 | 是 | 否（通过 Cleaner） |
| 适用场景 | 小数据量、频繁创建 | 大数据量、长期存在 |

### 5.2 使用建议

1. **选择合适的 Buffer 类型**
   ```java
   // 小数据量、临时使用
   ByteBuffer heapBuffer = ByteBuffer.allocate(1024);
   
   // 大数据量、需要与操作系统交互
   ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024 * 1024);
   ```

2. **正确使用 flip() 和 clear()**
   ```java
   // 写入数据后准备读取
   buffer.flip();
   
   // 读取完成后准备重新写入
   buffer.clear();
   
   // 部分读取后继续写入
   buffer.compact();
   ```

3. **避免频繁创建 Buffer**
   ```java
   // 不好的做法
   for (int i = 0; i < 1000; i++) {
       ByteBuffer buffer = ByteBuffer.allocate(1024);
       // 使用 buffer
   }
   
   // 好的做法：复用 Buffer
   ByteBuffer buffer = ByteBuffer.allocate(1024);
   for (int i = 0; i < 1000; i++) {
       buffer.clear();
       // 使用 buffer
   }
   ```

4. **注意直接内存的释放**
   ```java
   // DirectByteBuffer 会在 GC 时自动释放
   // 但可以通过以下方式显式释放（不推荐）
   DirectBuffer db = (DirectBuffer) directBuffer;
   Cleaner cleaner = db.cleaner();
   if (cleaner != null) {
       cleaner.clean();
   }
   ```

### 5.3 常见错误和陷阱

1. **忘记调用 flip()**
   ```java
   ByteBuffer buffer = ByteBuffer.allocate(1024);
   buffer.put("Hello".getBytes());
   // 错误：没有调用 flip()，position 在末尾，读取不到数据
   byte b = buffer.get();  // 会抛出 BufferUnderflowException
   
   // 正确做法
   buffer.flip();
   byte b = buffer.get();  // 正常读取
   ```

2. **clear() 不会清除数据**
   ```java
   ByteBuffer buffer = ByteBuffer.allocate(1024);
   buffer.put("Hello".getBytes());
   buffer.flip();
   buffer.clear();
   // 数据仍然存在，只是 position 和 limit 被重置
   // 如果需要清除数据，需要手动覆盖
   ```

3. **视图 Buffer 共享数据**
   ```java
   ByteBuffer buffer = ByteBuffer.allocate(1024);
   buffer.put("Hello".getBytes());
   buffer.flip();
   
   ByteBuffer view = buffer.duplicate();
   view.put(0, (byte) 'h');  // 修改视图
   // 原 buffer 的数据也被修改了
   ```

## 6. Buffer 性能优化

### 6.1 批量操作

```java
// 单个操作（慢）
for (int i = 0; i < 1000; i++) {
    buffer.put((byte) i);
}

// 批量操作（快）
byte[] data = new byte[1000];
for (int i = 0; i < 1000; i++) {
    data[i] = (byte) i;
}
buffer.put(data);
```

### 6.2 使用 slice() 避免复制

```java
// 不需要复制数据，创建子 Buffer
ByteBuffer subBuffer = buffer.slice();
subBuffer.position(10);
subBuffer.limit(20);
// 操作 subBuffer 不会影响原 buffer 的 position 和 limit
```

### 6.3 直接内存的合理使用

```java
// 对于需要与操作系统交互的场景，使用直接内存
// 例如：文件 I/O、网络 I/O
FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
ByteBuffer directBuffer = ByteBuffer.allocateDirect(8192);
channel.read(directBuffer);
```

## 7. 总结

### 7.1 核心要点

1. **Buffer 的四个核心属性**：capacity、position、limit、mark
2. **状态转换**：通过 `flip()`、`clear()`、`rewind()`、`compact()` 等方法切换状态
3. **两种实现**：`HeapByteBuffer`（堆内存）和 `DirectByteBuffer`（直接内存）
4. **视图机制**：可以创建只读视图、类型视图等，共享底层数据

### 7.2 实现原理要点

1. **HeapByteBuffer**：基于 Java 数组，受 GC 管理，访问速度相对较慢
2. **DirectByteBuffer**：使用堆外内存，通过 `Unsafe` 分配，使用 `Cleaner` 自动释放
3. **字节序**：影响多字节数据类型的存储和读取顺序
4. **视图共享**：多个视图 Buffer 共享底层数组，但各自维护独立的 position、limit、mark

### 7.3 使用建议

- 根据场景选择合适的 Buffer 类型
- 正确使用状态转换方法
- 避免频繁创建 Buffer，尽量复用
- 注意直接内存的使用和释放
- 使用批量操作提高性能

---

**参考资源**：
- [Java NIO Buffer 官方文档](https://docs.oracle.com/javase/8/docs/api/java/nio/Buffer.html)
- [Java NIO 教程](https://jenkov.com/tutorials/java-nio/buffers.html)

