---
title: String.intern() 方法
tags:
  - Java/字符串
  - 原理型
module: 09_字符串
created: 2026-04-18
---

# String.intern() 方法

## 核心结论

`String.intern()` 是一个 native 方法，作用是将字符串对象放入**字符串常量池**。如果常量池中已有相等的字符串，返回常量池中的引用；如果没有，将当前字符串加入常量池并返回常量池中的引用。

## 深度解析

### intern() 方法签名

```java
public native String intern();
```

### 工作原理

```
1. 检查字符串常量池中是否已存在 equals() 相等的字符串
2. 如果存在 → 返回常量池中的引用
3. 如果不存在 → 将当前字符串加入常量池，返回常量池中的引用
```

### JDK 6 vs JDK 7+ 的区别

```java
String s1 = new String("hello");  // 堆中创建对象
String s2 = s1.intern();          // 常量池中的引用

// JDK 6：常量池在永久代（方法区），intern() 会将字符串**复制**一份到常量池
// JDK 7+：常量池在堆中，intern() 会将堆中字符串的**引用**放入常量池
```

| 版本     | 常量池位置    | intern() 策略               |
| ------ | -------- | ------------------------- |
| JDK 6  | 永久代（方法区） | 如果常量池没有，**复制**一份字符串到永久代   |
| JDK 7+ | 堆中       | 如果常量池没有，将堆中对象的**引用**加入常量池 |

### 经典面试题

```java
String s1 = new String("hello") + new String("world");
String s2 = "helloworld";
System.out.println(s1 == s2);          // false（s1 在堆中，s2 在常量池）

String s3 = s1.intern();
System.out.println(s2 == s3);          // true（都是常量池中的引用）
System.out.println(s1 == s3);          // JDK 7+ true（s3 指向 s1 在堆中的引用）
                                        // JDK 6  false（s3 是永久代的副本）
```

### intern() 的使用场景

```java
// 场景：大量重复字符串时，减少内存占用
// 比如从数据库读取大量重复的城市名称
List<String> cities = ...;
List<String> pooled = cities.stream()
    .map(String::intern)
    .collect(Collectors.toList());
// 相同的城市名只保留一个对象，节省内存
```

### 注意事项

- `intern()` 会操作常量池，频繁调用可能有性能开销
- JDK 7+ 常量池在堆中，GC 可以回收常量池中无引用的字符串
- 不要滥用 `intern()`，只在字符串大量重复时使用

## 关联知识点
