---
title: 字符串常量池
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# 字符串常量池

## Q：什么是字符串常量池？

**A：** 字符串常量池（String Pool）是 JVM 中的一块特殊区域，用于存储字符串字面量和 `intern()` 加入的字符串。相同内容的字符串在常量池中只保留一份，目的是节省内存、提高效率。

- JDK 6：常量池在**永久代**（方法区）
- JDK 7+：常量池移到**堆**中，便于 GC 回收

## Q：字符串常量池中存的是什么？

**A：** 常量池中存储的是**字符串对象的引用**（JDK 7+），不是字符串内容本身（字符串内容在堆的对象中）：
- 字面量 `"hello"` 会在常量池中创建引用
- `new String("hello")` 在堆中创建对象，常量池中已有的 "hello" 引用不变

## Q：以下代码分别创建了几个对象？

**A：**
```java
String s1 = "hello";             // 0 或 1 个对象（常量池中没有则创建 1 个）
String s2 = "hello";             // 0 个对象（直接引用常量池已有的）
String s3 = new String("hello"); // 1 或 2 个对象（堆中 1 个 + 常量池可能 1 个）
String s4 = "hel" + "lo";        // 0 或 1 个对象（编译期合并，常量池中已有则 0）
```

## Q：常量池会被 GC 回收吗？

**A：**
- JDK 6：常量池在永久代，不受普通 GC 管理，可能 OOM
- JDK 7+：常量池在堆中，当常量池中的字符串没有任何强引用指向时，可以被 GC 回收

## Q：new String("hello") 创建了几个对象？

**A：** 创建了 **1 或 2 个对象**：
- **常量池中已有 "hello"**：只在**堆中**创建 1 个 String 对象
- **常量池中没有 "hello"**：在**常量池**创建 1 个 + 在**堆中**创建 1 个 = 2 个对象
堆中的 String 对象内部的 `char[]` 引用的是常量池中字符串的 `char[]`（共享），不会复制数组。

## Q：new String("he") + new String("llo") 创建了几个对象？

**A：** 创建了 **4 或 5 个对象**：
1. `"he"` 常量池中的对象（0 或 1 个）
2. `new String("he")` 堆中的对象（1 个）
3. `"llo"` 常量池中的对象（0 或 1 个）
4. `new String("llo")` 堆中的对象（1 个）
5. StringBuilder 拼接后 `toString()` 产生的新 String 对象（1 个）
注意：拼接结果 `"hello"` **不会自动进入常量池**，需要调用 `intern()` 才会放入。

## Q：以下代码输出什么？

**A：**
```java
String s1 = "hello";
String s2 = new String("hello");
System.out.println(s1 == s2);          // false（常量池 vs 堆）
System.out.println(s1 == s2.intern()); // true（都是常量池中的引用）
```
`s1` 指向常量池，`s2` 指向堆中对象，`s2.intern()` 返回常量池中的引用，与 `s1` 相同。
