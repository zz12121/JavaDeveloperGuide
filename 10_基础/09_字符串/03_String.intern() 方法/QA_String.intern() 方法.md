---
title: String.intern() 方法
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String.intern() 方法

## Q：String.intern() 方法的作用是什么？

**A：** `intern()` 是一个 native 方法，用于将字符串放入字符串常量池：
- 如果常量池中已存在值相等的字符串 → 返回常量池中的引用
- 如果常量池中不存在 → 将当前字符串加入常量池，返回常量池中的引用

## Q：JDK 6 和 JDK 7 的 intern() 有什么区别？

**A：** 核心区别在于常量池的位置不同：
- **JDK 6**：常量池在永久代（方法区），`intern()` 会将字符串**复制**一份到永久代
- **JDK 7+**：常量池移到堆中，`intern()` 只是将堆中对象的**引用**放入常量池，不再复制
```java
String s1 = new String("hello");
String s2 = s1.intern();
System.out.println(s1 == s2);
// JDK 6:  false（s2 是永久代的副本）
// JDK 7+: true（s2 和 s1 指向堆中同一个对象）
```

## Q：什么时候需要使用 intern()？

**A：** 当程序中存在大量**重复的字符串对象**时，使用 `intern()` 可以让这些重复字符串共享常量池中的同一个对象，减少内存占用。例如从数据库读取大量重复的城市名、用户名等。但不要滥用，频繁调用 `intern()` 本身也有性能开销。
