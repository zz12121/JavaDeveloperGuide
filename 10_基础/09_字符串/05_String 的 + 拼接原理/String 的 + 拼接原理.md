---
title: String 的 + 拼接原理
tags:
  - Java/字符串
  - 原理型
module: 09_字符串
created: 2026-04-18
---

# String 的 + 拼接原理

## 核心结论

Java 编译器对 `String +` 操作进行了优化：字面量拼接在**编译期**直接合并；包含变量的拼接在**编译期**自动转换为 `StringBuilder.append()`；但**循环内的 + 拼接**每次循环都创建新的 StringBuilder，性能差。

## 深度解析

### 编译期优化：字面量直接合并

```java
// 源码
String s = "hello" + " " + "world";

// 编译后（javap -c 查看）
String s = "hello world";  // 编译器直接合并为一个字符串
```

### 编译期优化：变量拼接转为 StringBuilder

```java
// 源码
String s = "hello" + name + "!";

// 编译后等价于
String s = new StringBuilder().append("hello").append(name).append("!").toString();
```

### 循环内的 + 拼接（性能陷阱）

```java
// ❌ 循环内 + 拼接：每次循环创建新的 StringBuilder
String s = "";
for (int i = 0; i < 1000; i++) {
    s += i;
}
// 编译后等价于：
String s = "";
for (int i = 0; i < 1000; i++) {
    s = new StringBuilder().append(s).append(i).toString();
}
// 每次循环：new StringBuilder → append → toString → new String
// 创建了约 3000 个临时对象（1000 StringBuilder + 1000 String + 原有的 s）

// ✅ 手动使用 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String s = sb.toString();
// 只创建 1 个 StringBuilder + 1 个最终 String
```

### 字符串常量 + final 变量

```java
final String prefix = "hello";
String s1 = prefix + " world";  // 编译期确定，等价于 "hello world"
// 因为 prefix 是 final，编译器知道其值不会变

String prefix2 = "hello";
String s2 = prefix2 + " world";  // 运行时拼接，创建 StringBuilder
// 因为 prefix2 不是 final，编译器不能确定其值
```

### JDK 9+ 的优化：invokedynamic

JDK 9 对字符串拼接引入了 `invokedynamic` + `StringConcatFactory`，不再每次都创建 StringBuilder：

```java
// JDK 9+ 编译后
String s = name + "!";
// 使用 invokedynamic 调用 StringConcatFactory.makeConcatWithConstants()
// JVM 可以策略选择：StringBuilder 方式或直接字节数组拼接
```

## 关联知识点

