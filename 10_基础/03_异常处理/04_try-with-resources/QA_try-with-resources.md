---
title: try-with-resources
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# try-with-resources

## Q1：什么是 try-with-resources？有什么好处？

**A：**
JDK 7 引入的语法糖，用于自动关闭资源。语法：
```java
try (Resource r = new Resource()) { ... }
```

好处：
1. 不需要手动写 finally 关闭资源，代码更简洁
2. 即使 try 块抛异常，资源也一定会被关闭
3. 解决了传统 finally 关闭时异常覆盖主异常的问题（通过 Suppressed Exception 机制）

---

## Q2：使用 try-with-resources 的类需要满足什么条件？

**A：**
资源类必须实现 `java.lang.AutoCloseable` 接口，重写 `close()` 方法。  
`java.io.Closeable` 是 `AutoCloseable` 的子接口（`close()` 签名声明抛 `IOException`）。

常见实现类：`InputStream`、`OutputStream`、`Reader`、`Writer`、`Connection`、`PreparedStatement` 等。

---

## Q3：try-with-resources 中如果 try 块和 close() 都抛出异常，哪个会被抛出？

**A：**
**主异常（try 块的异常）会被抛出**，`close()` 的异常被**抑制（suppressed）**，附加到主异常上，可通过 `e.getSuppressed()` 获取。

对比传统 try-finally：
```java
// finally 中异常会覆盖主异常，导致主异常丢失！
try {
    throw new Exception("主异常");
} finally {
    throw new Exception("finally 异常");  // 主异常丢失
}
```

try-with-resources 更安全，不会丢失主异常。

---

## Q4：多个资源的关闭顺序是什么？

**A：**
**与打开顺序相反**（后进先出）：

```java
try (
    ResourceA a = new ResourceA();   // 先打开
    ResourceB b = new ResourceB()    // 后打开
) { ... }
// 关闭：b.close() → a.close()
```

---

## Q5：JDK 9 对 try-with-resources 有什么改进？

**A：**
JDK 9 允许在括号外声明资源（变量必须是 effectively final）：
```java
InputStream in = new FileInputStream("file");
try (in) {  // 不需要重新声明
    // 使用 in
}
```
JDK 7/8 必须在括号内声明，JDK 9 更加灵活。
