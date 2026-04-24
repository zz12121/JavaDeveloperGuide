---
title: 四大核心函数式接口
tags:
  - Java/Lambda
  - 原理型
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# 四大核心函数式接口

## Q：四大核心函数式接口是什么？

**A：**

| 接口 | 方法 | 说明 |
|------|------|------|
| `Consumer<T>` | `void accept(T)` | 消费者：接收参数，无返回 |
| `Supplier<T>` | `T get()` | 供给者：无参数，返回结果 |
| `Predicate<T>` | `boolean test(T)` | 断言：接收参数，返回 boolean |
| `Function<T,R>` | `R apply(T)` | 函数：接收 T 返回 R |

## Q：Predicate 有哪些默认方法？

**A：**
- `and(Predicate)`：与操作，`p1.and(p2)` 等价于 `p1 && p2`
- `or(Predicate)`：或操作，`p1.or(p2)` 等价于 `p1 || p2`
- `negate()`：非操作，`p.negate()` 等价于 `!p`
```java
Predicate<Integer> isPositive = x -> x > 0;
Predicate<Integer> isEven = x -> x % 2 == 0;
isPositive.and(isEven).test(4);  // true：大于0且是偶数
```

## Q：Function 的 compose 和 andThen 有什么区别？

**A：**
- `compose(before)`：先执行 before，再执行当前函数（**从右到左**）
- `andThen(after)`：先执行当前函数，再执行 after（**从左到右**）
```java
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<Integer, Integer> addOne = x -> x + 1;

doubleIt.compose(addOne).apply(3);   // (3+1)*2 = 8
doubleIt.andThen(addOne).apply(3);   // (3*2)+1 = 7
```
