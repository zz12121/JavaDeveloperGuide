---
title: 构造器引用
tags:
  - Java/Lambda
  - 原理型
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# 构造器引用

## Q：什么是构造器引用？

**A：** 构造器引用使用 `类名::new` 语法，是方法引用的特殊形式。用于简化 Lambda 中创建对象的写法。根据函数式接口的参数个数自动匹配对应构造器：
```java
Supplier<User> s = User::new;                     // 匹配无参构造器
Function<String, User> f = User::new;              // 匹配一个参数构造器
BiFunction<String, Integer, User> bf = User::new;  // 匹配两个参数构造器
```

## Q：数组构造器引用怎么用？

**A：** 数组构造器引用语法为 `类型[]::new`，用于创建指定长度的数组：
```java
Function<Integer, String[]> factory = String[]::new;
String[] arr = factory.apply(10);  // 创建长度 10 的 String 数组

// 等价于
Function<Integer, String[]> factory2 = size -> new String[size];
```

## Q：构造器参数超过 2 个怎么办？

**A：** JDK 标准库只提供了 0~2 个参数的函数式接口（Supplier、Function、BiFunction）。超过 2 个参数需要自定义函数式接口：
```java
@FunctionalInterface
public interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}
TriFunction<String, Integer, String, User> factory = User::new;
```
