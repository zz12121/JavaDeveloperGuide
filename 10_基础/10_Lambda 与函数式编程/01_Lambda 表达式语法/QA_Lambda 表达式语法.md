---
title: Lambda 表达式语法
tags:
  - Java/Lambda
  - 原理型
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# Lambda 表达式语法

## Q：Lambda 表达式的基本语法是什么？

**A：** 基本格式为 `(参数列表) -> {方法体}`，可以逐步简化：

```java
// 完整形式
(String s) -> { return s.length(); }

// 省略参数类型
(s) -> { return s.length(); }

// 单参数省略括号
s -> s.length()

// 无参
() -> System.out.println("hello")
```

## Q：Lambda 和匿名内部类有什么区别？

**A：**
- **底层实现**：匿名内部类编译时生成 `.class` 文件；Lambda 通过 `invokedynamic` 运行时动态生成，不产生额外类文件
- **this 指向**：匿名内部类中的 `this` 指向匿名类自身；Lambda 中的 `this` 指向包含 Lambda 的外部类
- **访问限制**：匿名内部类可以访问外部类的实例变量；Lambda 和匿名内部类都可以访问 effectively final 的局部变量
- **接口要求**：匿名内部类可以实现任意接口（多个方法）；Lambda 只能用于函数式接口（一个抽象方法）

## Q：Lambda 底层是怎么实现的？

**A：** Lambda 编译后不是匿名内部类，而是使用 `invokedynamic` 指令。运行时由 `LambdaMetafactory` 动态生成函数式接口的实现类并实例化。首次调用时有生成开销，后续调用复用缓存的实现类。
