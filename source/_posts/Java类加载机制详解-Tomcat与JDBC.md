---
title: Java类加载机制详解：从Tomcat到JDBC
author: Sitoi
top: false
cover: false
toc: true
date: 2025-12-11 15:30:00
categories:
  - Java
  - JVM
tags:
  - ClassLoader
  - Tomcat
  - JDBC
  - 源码分析
summary: 本文深入解析 Java 类加载机制，重点探讨 Tomcat 如何利用自定义类加载器实现 Web 应用隔离与共享，以及 JDBC 驱动加载机制从手动注册到 SPI 自动加载的演进过程。
---

# Java类加载机制详解：从Tomcat到JDBC

在 Java 开发中，类加载机制（ClassLoader）是一个核心但容易被忽视的概念。本文将通过两个经典案例——Tomcat 的 Web 应用隔离和 JDBC 驱动的加载演进，深入剖析 Java 类加载机制的原理与应用。

## 目录

1. [Tomcat 类加载机制](#tomcat-类加载机制)
2. [JDBC 驱动加载演进](#jdbc-驱动加载演进)
3. [ServiceLoader 源码解析](#serviceloader-源码解析)
4. [深入思考：假如没有 TCCL](#深入思考假如没有-tccl)

---

## Tomcat 类加载机制

Tomcat 作为一款 Servlet 容器，需要解决一个核心问题：**如何实现不同的 Web 应用有相同的类，同时又能共享部分公共类？**

Tomcat 并没有简单地使用 JVM 默认的单一应用类加载器 (AppClassLoader) 来加载所有的 Web 应用，而是设计了一个精妙的分层类加载器结构。

### 1. 类加载器层次结构

Tomcat 的类加载器从上到下依次为：

| 加载器 | 负责加载的内容 | 加载器类型 |
| :--- | :--- | :--- |
| **Bootstrap** | JVM 核心类库 (rt.jar, etc.) | JVM 内置 |
| **System** | Tomcat 启动时使用的类（如 Catalina.jar） | JVM 内置 (AppClassLoader) |
| **Common** | 所有 Web 应用共享的库（如 lib/*.jar） | Tomcat 定制 |
| **Webapp** | 单个 Web 应用的类和资源 | Tomcat 定制 |

### 2. 实现代码级别的隔离与共享

Tomcat 实现隔离和共享的关键在于它的 **“倒置”双亲委派模型**（WebappClassLoader 的特殊委派机制）。

#### WebappClassLoader 的作用
每个部署到 Tomcat 的 Web 应用（Context）都会创建一个独立的 `WebappClassLoader` 实例。
- **隔离**：确保应用 A 加载的类不会被应用 B 访问，即使类名和包名相同。
- **热部署**：销毁一个 WebappClassLoader 实例即可卸载整个应用，而不影响 JVM 其他部分。

#### 场景一：实现“相同的类，不同的版本” (隔离)

假设应用 A 和应用 B 都依赖 `StringUtils`，但 A 使用 v3.5，B 使用 v3.9。

**加载流程：**
1. **应用 A** 引用 `StringUtils`。
2. `WebappClassLoader_A` **首先在自己的仓库中查找** (`/WEB-INF/lib/commons-lang3-3.5.jar`)。
3. 找到后直接加载 v3.5 的类。
4. **应用 B** 引用 `StringUtils`。
5. `WebappClassLoader_B` **首先在自己的仓库中查找** (`/WEB-INF/lib/commons-lang3-3.9.jar`)。
6. 找到后直接加载 v3.9 的类。

**结果**：JVM 中存在两个不同的 `StringUtils` 类实例（`@hashA` 和 `@hashB`），彼此完全隔离。

#### 场景二：实现“共享相同的类” (共享)

Tomcat 必须共享 Java 核心库（`java.*`）以及 Tomcat 自身所需的类（如 `servlet-api.jar`）。

**加载流程：**
1. 应用 A 引用 `javax.servlet.http.HttpServlet`。
2. `WebappClassLoader` 对特定的 Java API 包（如 `java.*`, `javax.servlet.*`）**强制走双亲委派**。
3. 委托给 `Common ClassLoader` -> `System` -> `Bootstrap`。
4. 最终由上层加载器加载，保证所有应用使用的是同一个 `HttpServlet` 类。

### 3. 源码解析：WebappClassLoader.loadClass

`WebappClassLoaderBase` 重写了 `loadClass` 方法，颠倒了默认的查找顺序：

```java
// 伪代码，展示 WebappClassLoader 的核心加载逻辑
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    Class<?> clazz = null;

    // 1. 检查是否是需要委派给父加载器的核心类 (强制委派)
    //    比如 java.lang.*, javax.servlet.* 等
    if (mustDelegateToParent(name)) {
        try {
            clazz = super.loadClass(name, resolve); // 走标准双亲委派
        } catch (ClassNotFoundException e) {
            // 继续下一步
        }
    }

    // 2. 在应用自己的仓库中查找 (打破双亲委派)
    //    先找 /WEB-INF/classes 或 /WEB-INF/lib
    if (clazz == null) {
        clazz = findClass(name); 
    }

    // 3. 如果还没找到，才委托给 Common ClassLoader
    if (clazz == null) {
        if (!mustDelegateToParent(name)) { 
            try {
                // 委托给 Common ClassLoader 及其父加载器
                clazz = super.loadClass(name, resolve); 
            } catch (ClassNotFoundException e) {
                // ... 抛出异常 ...
            }
        }
    }
    
    return clazz;
}
```

---

## JDBC 驱动加载演进

JDBC 驱动的加载机制演变，完美诠释了 Java SPI (Service Provider Interface) 和 **线程上下文类加载器 (TCCL)** 的作用。

### 机制对比

| 特性 | TCCL 出现前 (JDBC < 4.0) | TCCL 出现后 (JDBC ≥ 4.0) |
| :--- | :--- | :--- |
| **加载方式** | 显式加载 (`Class.forName`) | 自动发现 (SPI/ServiceLoader) |
| **发起方** | 应用程序代码 | `java.sql.DriverManager` 核心库 |
| **加载器** | AppClassLoader | Thread Context ClassLoader (TCCL) |

### 1. 阶段一：手动加载 (Class.forName)

在 JDBC 4.0 之前，开发者必须显式加载驱动：

```java
// 应用程序代码
try {
    // 1. 显式加载驱动类 -> 触发静态代码块
    Class.forName("com.mysql.jdbc.Driver"); 
} catch (ClassNotFoundException e) { ... }

// 2. 建立连接
Connection conn = DriverManager.getConnection(url, user, password);
```

**原理：**
- `Class.forName()` 使用调用者的类加载器（即 AppClassLoader）加载驱动类。
- 驱动类内部包含静态代码块，加载时自动向 `DriverManager` 注册自己：

```java
public class Driver implements java.sql.Driver {
    static {
        try {
            // 主动向 DriverManager 注册
            DriverManager.registerDriver(new Driver());
        } catch (SQLException E) { ... }
    }
}
```

### 2. 阶段二：自动加载 (ServiceLoader + TCCL)

JDBC 4.0 引入了 SPI 机制，不再需要显式调用 `Class.forName`。

```java
// 应用程序代码
Connection conn = DriverManager.getConnection(url, user, password);
```

**原理：**
1. `DriverManager.getConnection` 内部调用 `loadInitialDrivers()`。
2. `loadInitialDrivers` 使用 `ServiceLoader` 查找驱动：
   ```java
   ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
   ```
3. `ServiceLoader` 使用 **TCCL**（通常是 AppClassLoader）去查找 `META-INF/services/java.sql.Driver` 文件。
4. 找到后，利用 TCCL 加载驱动类，触发静态代码块注册。

---

## ServiceLoader 源码解析

`java.util.ServiceLoader` 是 JDK 核心库的一部分（`rt.jar` 或 `java.base` 模块），它通过 TCCL 打破了双亲委派模型。

### 核心代码

```java
// java.util.ServiceLoader.java

public static <S> ServiceLoader<S> load(Class<S> service) {
    // 1. 获取当前线程的上下文类加载器！
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    
    // 2. 将加载任务委托给内部构造函数，传入 TCCL
    return ServiceLoader.load(service, cl);
}
```

### 为什么需要 TCCL？
- `ServiceLoader` 类本身由 Bootstrap ClassLoader 加载。
- 假如它使用自己的加载器去查找驱动，只能在 JDK 核心库中找，无法看到应用 classpath 下的第三方驱动包（如 `mysql-connector.jar`）。
- **TCCL 充当了桥梁**：它允许核心库代码（ServiceLoader）“借用”应用层类加载器（AppClassLoader）来加载类。

---

## 深入思考：假如没有 TCCL

如果我们**既不使用 `Class.forName`（手动加载），也不使用 TCCL（自动加载）**，会发生什么？

### 实验代码
```java
// 假设环境：删除了 Class.forName，且将 TCCL 置空
Thread.currentThread().setContextClassLoader(null);
Connection conn = DriverManager.getConnection(url, user, password);
```

### 结果：数据库连接失败
抛出异常：`java.sql.SQLException: No suitable driver found`。

### 原因分析
1. **ServiceLoader 回退**：
   当 TCCL 为 `null` 时，`ServiceLoader` 会尝试使用系统类加载器（System ClassLoader）或者回退到加载 `ServiceLoader` 自身的加载器（Bootstrap ClassLoader）。
   
2. **可见性问题**：
   - Bootstrap ClassLoader 只能看到 `rt.jar`，看不到 `mysql-connector.jar`。
   - System ClassLoader (如果回退到这里) 虽然通常能看到，但在某些复杂环境（如 OSGi、Web 容器）中可能也无法准确找到应用路径。
   
3. **注册失败**：
   驱动类无法被加载 -> 静态代码块未执行 -> `DriverManager` 中没有注册任何驱动 -> 连接失败。

### 结论
在 Java 类加载体系中，核心库（Bootstrap 加载）要调用应用代码（AppClassLoader 加载），必须依靠 **显式回调**（如 `Class.forName`）或者 **TCCL 隐式委派**。二者必居其一，否则双亲委派模型的隔离性将导致“父”无法看到“子”的类。

