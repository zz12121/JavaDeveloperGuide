---
title: JDK 9新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口私有方法

## 核心结论

JDK 9 允许接口中定义 **private 方法**（包括 `private` 实例方法和 `private static` 方法），用于在接口内部实现代码复用，避免 default 方法之间出现重复代码。private 方法**不会暴露给实现类**，也不能被继承。

---

## 深度解析

### 1. 为什么需要接口私有方法？

JDK 8 引入 default 方法后，如果多个 default 方法有重复逻辑，需要一个地方放公共代码。但不能用 default 方法（会暴露给实现类），也不能用 static 方法（无法访问实例上下文）。JDK 9 的 private 方法解决了这个问题。

```java
public interface Logging {
    // 两个 default 方法有重复逻辑
    default void logInfo(String msg) {
        formatAndLog("INFO", msg);  // 重复
    }

    default void logError(String msg) {
        formatAndLog("ERROR", msg); // 重复
    }

    // JDK 9 之前：重复代码无法提取
    // JDK 9 之后：提取为 private 方法
    private void formatAndLog(String level, String msg) {
        System.out.println("[%s] %s - %s".formatted(
            LocalDateTime.now(), level, msg));
    }
}
```

### 2. 语法规则

```java
public interface MyInterface {
    // private 实例方法：可被 default 方法调用
    private void instanceHelper() {
        System.out.println("私有实例方法");
    }

    // private static 方法：可被 default 和 static 方法调用
    private static void staticHelper() {
        System.out.println("私有静态方法");
    }

    default void method1() {
        instanceHelper(); // ✅
        staticHelper();   // ✅
    }

    static void method2() {
        staticHelper();   // ✅
        // instanceHelper(); // ❌ 静态方法不能调用实例方法
    }
}
```

### 3. 关键规则

| 规则 | 说明 |
|------|------|
| 访问修饰符 | 只能是 `private`，不能用 `protected` 或 `public` |
| 调用范围 | 仅接口内部可调用 |
| 实现类访问 | ❌ 不可访问，不可重写 |
| private 实例方法 | 可被 default 方法和 private 实例方法调用 |
| private static 方法 | 可被 default 方法、static 方法、private 方法调用 |

### 4. 接口方法访问级别演进

| JDK 版本 | 接口支持的方法类型 |
|----------|-------------------|
| JDK 7 及之前 | 只有 `public abstract` 方法 |
| JDK 8 | 新增 `default` 和 `public static` 方法 |
| JDK 9 | 新增 `private` 和 `private static` 方法 |

---

## 代码示例

```java
public interface Validator {
    // public 接口方法
    boolean validate(String input);

    // default 方法
    default void validateAndPrint(String input) {
        boolean result = doValidate(input);
        System.out.println(result ? "验证通过" : "验证失败");
    }

    default void validateOrFail(String input) {
        if (!doValidate(input)) {
            throw new IllegalArgumentException("验证失败: " + input);
        }
    }

    // private 方法：提取公共验证逻辑
    private boolean doValidate(String input) {
        return input != null && !input.isBlank() && input.length() <= 100;
    }
}
```

---


# 集合工厂方法

## 核心结论

JDK 9 为 `List`、`Set`、`Map` 接口新增了 `of()` 系列工厂方法，用于快速创建**不可变集合**。相比 `Arrays.asList()` 和 `Collections.unmodifiableList()`，`of()` 更简洁且创建的集合**天然不可变**（null 值、增删操作都会抛异常）。

---

## 深度解析

### 1. 基本用法

```java
// List.of()
var list1 = List.of("A", "B", "C");
var emptyList = List.of();

// Set.of()
var set1 = Set.of("A", "B", "C");
var emptySet = Set.of();

// Map.of()
var map1 = Map.of("key1", "value1", "key2", "value2");
var emptyMap = Map.of();

// Map.ofEntries() — 适合更多键值对
var map2 = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
);
```

### 2. 不可变性

```java
List<String> list = List.of("A", "B", "C");
list.add("D");      // UnsupportedOperationException
list.set(0, "X");   // UnsupportedOperationException
list.remove(0);     // UnsupportedOperationException
```

> 创建后**不可增删改**，任何修改操作都抛 `UnsupportedOperationException`。

### 3. null 值限制

```java
List.of("A", null, "C");  // NullPointerException
Set.of(null);             // NullPointerException
Map.of("key", null);      // NullPointerException
```

> `of()` **不允许 null 元素**。如需允许 null，使用 `ArrayList` 或 `Collections.singletonList()`。

### 4. Set 的去重

```java
Set.of("A", "B", "A");  // IllegalArgumentException: 重复元素
```

> `Set.of()` 会在创建时检查重复，发现重复立即抛 `IllegalArgumentException`。

### 5. 与 Arrays.asList() 对比

| 对比项 | `Arrays.asList()` | `List.of()` |
|--------|-------------------|-------------|
| 可变性 | 固定大小，但元素可 set | 完全不可变（元素也不可 set） |
| null 元素 | 允许 | 不允许 |
| null 数组参数 | 允许 | 不允许 |
| JDK 版本 | JDK 1.2 | JDK 9 |
| 与原数组关系 | 共享底层数组 | 独立，不受影响 |

### 6. 实现原理

```java
// 不同元素数量使用不同实现类，避免数组开销
List.of()                  // ListN.EMPTY（空实例）
List.of("A")              // ListN（1个元素）
List.of("A", "B", "C")    // ListN（N个元素，内部用数组存储）
```

> `List.of()` 返回的是不可变的 `ListN` 实例，实现了序列化等接口。

---

## 代码示例

```java
// JDK 9 之前
List<String> names = Collections.unmodifiableList(
    Arrays.asList("Alice", "Bob", "Charlie")
);

// JDK 9 之后
List<String> names = List.of("Alice", "Bob", "Charlie");

// 常见使用场景：方法返回常量集合
public List<String> getSupportedFormats() {
    return List.of("JSON", "XML", "YAML");
}
```

---

---

# Java 平台模块系统（JPMS）

## 核心结论

JPMS（Java Platform Module System，JEP 261）是 JDK 9 引入的最重要特性，通过 `module-info.java` 将 Java 项目拆分为**模块**，实现显式依赖声明、强封装（`exports` / `opens`）和可靠配置。JDK 9 本身也按模块化重构为约 95 个模块。JPMS 解决了类路径（ClassPath）的传递依赖不透明、 JAR 地狱等问题。

> **面试重点**：模块化是 Java 架构层面的重大变更，虽然普通业务代码感知不强，但涉及 JDK 9+ 运行时、Spring Boot 3.x 最小镜像打包时，必须理解 `requires`、`exports`、`opens` 的区别。

---

## 深度解析

### 1. 为什么需要模块化？

**ClassPath 的问题**：

```java
// 应用依赖 JAR A（需要 X 库 1.0 版本）
// 应用依赖 JAR B（需要 X 库 2.0 版本）
// → ClassPath 对版本不敏感，运行时可能加载错误的类
// → 没有任何机制阻止"非法访问"——所有 public 类都可以被访问
```

**JPMS 的解决方案**：

```java
// module-info.java：声明模块的依赖和导出
module com.myapp {
    requires com.example.library;    // 明确依赖哪个模块
    exports com.myapp.api;          // 只导出公开 API
    opens com.myapp.internal;       // 仅对反射开放
}
```

### 2. 基本语法：module-info.java

```java
// 模块声明文件必须放在 src/main/java/ 根目录
module com.example.myapp {
    // requires：声明模块依赖（传递依赖自动继承）
    requires com.google.guava;
    requires com.fasterxml.jackson;    // 必需

    // requires static：编译时依赖，运行时不需要
    requires static lombok;            // 仅编译期需要

    // exports：导出包，仅导出的 public 类型可被其他模块访问
    exports com.example.api;
    exports com.example.dto;

    // exports ... to：精确控制哪个模块可以访问（限定导出）
    exports com.example.internal to com.example.plugin;

    // opens / opens ... to：开放包供反射访问（运行时）
    opens com.example.service;         // 对所有模块开放反射
    opens com.example.entity to com.example.mapper;  // 仅对特定模块开放

    // provides ... with：服务提供者（ServiceLoader 机制）
    provides PaymentGateway with AlipayGateway;

    // uses：服务消费者
    uses PaymentGateway;
}
```

### 3. 四个关键字的作用域

| 关键字 | 作用 | 可见性 |
|--------|------|--------|
| `requires` | 声明模块依赖 | 编译期 + 运行时 |
| `exports` | 导出包（仅导出的 public 类可访问）| 编译期 + 运行时 |
| `opens` | 开放包供反射（运行时）| 仅运行时（Spring/Jackson 反射）|
| `uses` | 声明服务接口依赖 | 编译期 |

### 4. 导出 vs 开放：为什么需要 opens？

```java
// 场景：Spring 依赖注入需要反射访问 private 字段
// 场景：Jackson 反序列化需要反射设置属性

// exports：编译期和运行时都只能访问 public 类型
module A {
    exports com.example.entity;
}

class User {                    // public class
    private String name;        // private 字段
}

// 在其他模块中：
User u = new User();            // ✅ 通过 exports 可以访问
u.name = "test";                // ❌ 编译错误：private 字段
u.getClass().getDeclaredField("name").set(u, "test");  // ❌ 运行时错误

// opens：允许运行时反射访问，包括 private 成员
module A {
    opens com.example.entity;
}

u.getClass().getDeclaredField("name").set(u, "test");  // ✅ 运行时允许
```

### 5. ServiceLoader 机制

JPMS 将 JDK 6 的 `ServiceLoader` 纳入模块系统，实现模块间的**可插拔服务发现**：

```java
// 模块 com.example.spi 中定义了服务接口
module com.example.spi {
    exports com.example.spi;
}

package com.example.spi;
public interface Cipher {
    String encrypt(String text);
}

// 模块 com.example.implA 提供了实现
module com.example.implA {
    requires com.example.spi;
    provides com.example.spi.Cipher with com.example.implA.AESCipher;
}

// 模块 com.example.app 使用服务
module com.example.app {
    requires com.example.spi;
    uses com.example.spi.Cipher;   // 声明使用 Cipher
}

// 运行时代码
ServiceLoader<Cipher> loader = ServiceLoader.load(Cipher.class);
Cipher cipher = loader.findFirst().orElseThrow();
```

### 6. 模块化 JDK 的架构

JDK 9 将 6000+ 个类重组为约 95 个模块：

```java
// JDK 9+ 常用模块
java.base        // 核心基础（Object/String/Collection 等），所有模块隐式依赖
java.logging      // 日志
java.sql          // JDBC
java.naming       // JNDI
java.management   // JMX 管理
jdk.crypto.ec     // 椭圆曲线加密（可移除，减小镜像体积）

// 不再需要的模块（JDK 9+）
java.xml.ws       // JAX-WS（已移除）
java.corba        // CORBA（已移除）
java.transaction  // JTA（已移除）
```

> **面试亮点**：Spring Boot 3.x 使用 GraalVM Native Image 构建最小镜像时，需要显式声明哪些 JDK 模块被使用，通过 `jlink --add-modules` 可以创建只包含必要模块的运行时镜像，大幅减小体积。

### 7. 模块路径 vs 类路径

| 维度 | 类路径（ClassPath）| 模块路径（Module Path）|
|------|-----------------|---------------------|
| 分辨率 | JAR 名（无版本感知）| 模块名（精确依赖）|
| 可见性 | 所有 JAR 的 public 类型 | 只有 `exports` 的类型 |
| 传递依赖 | 全部自动包含 | `requires` 才传递 |
| 反射 | 无限制 | 需要 `opens` |
| 依赖冲突 | 无感知 | 可检测（同一模块不同版本）|

---

## 代码示例

```bash
# 编译带模块的项目
javac -d out --module-source-path src $(find src -name "*.java")

# 运行模块化应用
java --module-path out -m com.example.app/com.example.app.Main

# 创建自定义 JRE（仅包含需要的模块）
jlink --add-modules java.base,java.sql,java.logging \
    --output my-jre \
    --launcher app=com.example.app

# 查看模块依赖
java --list-modules

# 分析模块依赖
jdeps --module-path out --module com.example.app
```

```java
// 运行时检查当前模块
Module current = MyClass.class.getModule();
System.out.println(current.getName());  // com.example.myapp
System.out.println(current.isExported("com.example.api"));  // true/false
```

---

## 关联知识点


