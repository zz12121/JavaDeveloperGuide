---
title: String 的 + 拼接原理
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String 的 + 拼接原理

## Q：String 的 + 拼接底层是怎么实现的？

**A：** 分两种情况：
1. **字面量拼接**（编译期优化）：`"a" + "b"` → 直接合并为 `"ab"`，不创建中间对象
2. **变量拼接**：编译器自动转换为 `new StringBuilder().append(a).append(b).toString()`

```java
String s = "hello" + name;
// 编译后等价于
String s = new StringBuilder().append("hello").append(name).toString();
```

## Q：循环内的 String + 拼接有什么问题？

**A：** 循环内每次 `+=` 都会创建新的 `StringBuilder` 和 `String` 对象（共约 3 个对象/次），导致大量临时对象和 GC 压力：
```java
String s = "";
for (int i = 0; i < 1000; i++) {
    s += i;  // 每次循环：new StringBuilder → append → toString
}

// 应改为
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // 只操作一个 StringBuilder
}
```

## Q：final 变量和普通变量拼接有区别吗？

**A：** 有区别。`final` 变量在编译期值确定，编译器会将其视为常量直接合并：
```java
final String a = "hello";    // 编译期常量
String s1 = a + " world";    // 编译期合并为 "hello world"

String b = "hello";          // 普通变量
String s2 = b + " world";    // 运行时用 StringBuilder 拼接
```
