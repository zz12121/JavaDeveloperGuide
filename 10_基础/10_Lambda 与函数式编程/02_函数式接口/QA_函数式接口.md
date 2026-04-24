---
title: 函数式接口
tags:
  - Java/Lambda
  - 原理型
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# 函数式接口

## Q：什么是函数式接口？

**A：** 函数式接口是**只包含一个抽象方法**的接口（可以包含 default 方法、static 方法、Object 类方法）。函数式接口是 Lambda 表达式的**目标类型**。

```java
@FunctionalInterface
public interface MyFunc {
    String process(String input);  // 唯一的抽象方法
}
// 使用
MyFunc f = s -> s.toUpperCase();
```

## Q：@FunctionalInterface 注解是必须的吗？

**A：** 不是。只要接口满足"一个抽象方法"的条件，它就是函数式接口，可以用于 Lambda。`@FunctionalInterface` 只是让编译器帮你检查，如果接口不满足条件会编译报错，防止后续维护时意外新增抽象方法。

## Q：函数式接口中能不能有 default 方法？

**A：** 可以。`default` 方法和 `static` 方法都不算抽象方法，不影响函数式接口的身份。从 Object 类继承的 public 方法（toString、equals、hashCode）也不算。只要**非 Object、非 default/static 的抽象方法只有一个**就是函数式接口。

## Q：JDK 中有哪些常用的函数式接口？

**A：**
- `Runnable`：`void run()` — 无参无返回
- `Callable<V>`：`V call()` — 无参有返回
- `Comparator<T>`：`int compare(T, T)` — 比较
- `Consumer<T>`：`void accept(T)` — 消费
- `Supplier<T>`：`T get()` — 供给
- `Predicate<T>`：`boolean test(T)` — 断言
- `Function<T,R>`：`R apply(T)` — 转换
