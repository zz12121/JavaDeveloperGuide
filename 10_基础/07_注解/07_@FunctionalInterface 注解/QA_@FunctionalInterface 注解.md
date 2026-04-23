---
title: "@FunctionalInterface 注解"
tags:
  - Java/注解
  - 原理型
  - 问答
module: 07_注解
created: 2026-04-18
---

# @FunctionalInterface 注解

## Q：@FunctionalInterface 注解的作用是什么？

**A：** `@FunctionalInterface` 用于标记一个接口为函数式接口，触发编译器检查：
- 接口必须**只有一个抽象方法**，否则编译报错
- 它只是一个**信息性注解**，不写也不影响使用，写了是为了让编译器帮助检查

```java
@FunctionalInterface
public interface MyFunc {
    void run();  // 只能有一个抽象方法
}
```

## Q：函数式接口中可以有 default 方法和 static 方法吗？

**A：** 可以。`default` 方法和 `static` 方法不算抽象方法，不影响函数式接口的身份。此外，从 `Object` 类继承的 `toString()`、`equals()`、`hashCode()` 方法也不算抽象方法。函数式接口的"一个抽象方法"指的是**非 Object 方法且非 default/static 的抽象方法**。

## Q：@FunctionalInterface 不写会怎样？

**A：** 不影响。只要接口满足"只有一个抽象方法"的条件，它就是一个函数式接口，可以用于 Lambda 表达式。`@FunctionalInterface` 只是让编译器帮你做检查，防止后续维护时意外增加抽象方法导致破坏 Lambda 的使用。
