---
title: JDK 9新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口私有方法

## Q1：JDK 9 为什么要在接口中引入 private 方法？

**A**：JDK 8 的 default 方法解决了接口功能扩展问题，但当多个 default 方法有重复代码时，需要一个内部复用机制。不能把重复代码提取为新的 default 方法（会暴露给实现类），也不能用 static 方法（无法访问实例上下文）。private 方法正好填补了这个空白。

---

## Q2：接口中的 private 方法和 private static 方法有什么区别？

**A**：

| 对比项 | `private` 方法 | `private static` 方法 |
|--------|---------------|---------------------|
| 归属 | 实例级别 | 类级别 |
| 调用者 | default 方法、其他 private 实例方法 | default 方法、static 方法、private 方法 |
| 能否调用 default 方法 | ❌ 不能 | ❌ 不能 |
| 能否访问实例相关内容 | 可以（虽然接口没有实例字段） | 不能 |

---

## Q3：实现类能访问接口中的 private 方法吗？

**A**：不能。private 方法完全属于接口内部实现细节，实现类无法看到、无法调用、也无法重写。这和类的 private 方法一样遵循封装原则。

---

## Q4：private 方法在接口中的使用场景是什么？

**A**：主要用于多个 default 方法之间的**代码复用**。例如：

```java
interface Logger {
    default void info(String msg) { log("INFO", msg); }
    default void error(String msg) { log("ERROR", msg); }
    default void warn(String msg) { log("WARN", msg); }

    // 三个 default 方法共享同一个 private 实现
    private void log(String level, String msg) {
        System.out.println("[%s] %s: %s".formatted(level, timestamp(), msg));
    }
}
```


---

# 集合工厂方法

## Q1：List.of() 和 Arrays.asList() 有什么区别？

**A**：

| 对比项 | `List.of()` | `Arrays.asList()` |
|--------|-------------|-------------------|
| 可变性 | 完全不可变 | 固定大小，但元素可 set |
| null 元素 | ❌ 不允许 | ✅ 允许 |
| 与原数组关系 | 独立 | 共享底层数组 |
| JDK 版本 | JDK 9 | JDK 1.2 |
```java
var a = Arrays.asList("A", "B");
a.set(0, "X");   // ✅ 可以修改元素

var b = List.of("A", "B");
b.set(0, "X");   // ❌ UnsupportedOperationException
```

---

## Q2：List.of() 创建的集合可以修改吗？

**A**：完全不可修改。不能增（add）、不能删（remove）、不能改（set），任何修改操作都会抛 `UnsupportedOperationException`。如果需要可变集合，可以包装一下：
```java
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
```

---

## Q3：List.of() 为什么不允许 null？

**A**：这是 JDK 的设计决策。不可变集合如果允许 null 元素，会导致 `contains(null)` 等方法的行为需要额外处理（返回 true 还是 false？）。禁止 null 可以简化实现，避免歧义。如需 null，使用 `ArrayList` 或 `new CopyOnWriteArrayList<>()`。

---

## Q4：Map.of() 最多支持多少个键值对？

**A**：`Map.of()` 重载方法最多支持 **10 个键值对**（5 对参数版本）。超过 10 个使用 `Map.ofEntries(Map.entry(...), ...)` 或 `Map.copyOf(map)`。

---

# Java 平台模块系统（JPMS）

## Q5：什么是 JPMS？为什么需要模块化？

**A**：JPMS（Java Platform Module System，JEP 261）是 JDK 9 引入的模块系统，通过 `module-info.java` 文件声明模块的依赖、导出和开放权限。

**解决的问题**：

| 问题 | ClassPath 时代 | JPMS 模块系统 |
|------|--------------|-------------|
| 依赖透明性 | JAR 依赖不透明，传递依赖全包含 | `requires` 显式声明 |
| 版本冲突 | 类路径无法感知版本，同名类冲突 | 同一模块不能有两个版本 |
| 可见性 | 所有 public 类都能被访问 | `exports` 控制可见性 |
| 反射权限 | 无限反射 | 需要 `opens` 才允许运行时反射 |

**JDK 9 本身也模块化了**：原来 6000+ 个类被组织成约 95 个模块，如 `java.base`、`java.sql`、`jdk.crypto.ec`，可以通过 `java --list-modules` 查看。

---

## Q6：`exports` 和 `opens` 有什么区别？

**A**：

- `exports`：编译期和运行时都可以访问导出包中的 **public 类型**（不包括 private 成员）
- `opens`：允许运行时**反射访问**包内所有成员（包括 private）

```java
module com.example {
    exports com.example.api;   // 其他模块可以访问 public 类
    opens com.example.entity;  // 其他模块可以通过反射访问 private 字段
}

// 其他模块中：
User user = new User();              // ✅ exports 允许
user.name = "test";                  // ❌ 编译错误：name 是 private

user.getClass().getDeclaredField("name").set(user, "test");
// exports：编译错误（name 是 private）
// opens：运行时 OK（Spring 依赖注入/Jackson 反序列化依赖此特性）
```

> **面试总结**：`exports` 控制编译期可见性，`opens` 控制运行时反射可见性。Spring Boot 和 Jackson 能在 JDK 9+ 运行就是因为它们使用了 `opens`（JDK 9 前是无限制反射，JDK 9 后需要显式声明）。

---

## Q7：`requires` 和 `uses` 有什么区别？

**A**：

| 关键字 | 作用 | 用途 |
|--------|------|------|
| `requires` | 声明编译期和运行期的模块依赖 | 正常依赖 |
| `requires static` | 仅声明编译期依赖，运行时不需要 | Lombok、编译期注解处理器 |
| `uses` | 声明使用某个服务接口（ServiceLoader） | SPI 扩展点 |

```java
module com.example.app {
    // 正常运行依赖
    requires com.google.guava;

    // 编译期依赖（运行时不需要）
    requires static lombok;

    // 服务消费者：需要 PaymentGateway 接口，但不直接依赖实现
    uses com.example.spi.PaymentGateway;
}
```

---

## Q8：JDK 9 之前哪些模块被移除了？

**A**：JDK 9 重构时移除了以下已过时的模块：

| 被移除的模块 | 原功能 | 替代方案 |
|-------------|--------|---------|
| `java.xml.ws` | JAX-WS（Web Service） | `jakarta.xml.ws`（独立 JAR）|
| `java.corba` | CORBA / RMI-IIOP | 第三方库 |
| `java.transaction` | JTA（Java Transaction API）| `jakarta.transaction` |
| `java.activation` | JavaBeans Activation | 第三方库 |
| `java.xml.bind` | JAXB（XML Binding）| `jakarta.xml.bind` |

> **实际影响**：如果项目从 JDK 8 迁移到 JDK 11+，使用 JAXB 的代码会编译失败，需要手动添加 JAXB 依赖。

---

## Q9：Spring Boot 在 JDK 9+ 是如何解决 `opens` 问题的？

**A**：Spring Framework 5.x 大量使用反射（依赖注入、AOP、Bean 创建），JDK 9 后通过以下方式兼容：

1. **运行时自动打开模块**（`--add-opens` JVM 参数）：
   ```bash
   java --add-opens java.base/java.lang=ALL-UNNAMED \
        --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
        -jar myapp.jar
   ```

2. **Spring Boot 3.x 提供 GraalVM 原生支持**：通过 `native-image` 编译时直接分析反射使用并生成配置，无需运行时 `--add-opens`。

> **面试总结**：Spring Boot 2.x 通过 `--add-opens` 参数解决，Spring Boot 3.x 通过 GraalVM 原生镜像解决。

**A**：`Map.of()` 重载方法最多支持 **10 个键值对**（5 对参数版本）。超过 10 个使用 `Map.ofEntries(Map.entry(...), ...)` 或 `Map.copyOf(map)`。
