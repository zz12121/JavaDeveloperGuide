---
title: Stream的创建
tags:
  - Java/Stream
  - 原理型
  - 问答
module: 11_Stream API
created: 2026-04-18
---

# Stream的创建
## Q1：Stream 有哪些创建方式？

**A**：主要有以下几种：
1. **Collection.stream() / parallelStream()** — 集合转流，最常用
2. **Arrays.stream()** — 数组转流，支持基本类型特化流
3. **Stream.of()** — 直接指定元素，但不支持基本类型数组自动拆箱
4. **Stream.generate()** — 无限流，传入 Supplier
5. **Stream.iterate()** — 无限流，传入种子和递推函数
6. **IntStream.range() / rangeClosed()** — 数值范围流

> **易错点**：`Stream.of(int[])` 返回的是 `Stream<int[]>`（一个包含数组的流），而非 `IntStream`。基本类型数组要用 `Arrays.stream()`。

---

## Q2：Stream.of() 和 Arrays.stream() 有什么区别？

**A**：
- **`Stream.of(T... values)`**：接收可变参数。对于对象数组正常工作，但对于基本类型数组 `int[]`，整个数组会被当作**一个元素**，产生 `Stream<int[]>`。
- **`Arrays.stream(int[] array)`**：有专门的重载方法，能正确识别基本类型数组，返回 `IntStream`。
```java
int[] arr = {1, 2, 3};
Stream.of(arr).count();       // 1（整个数组是一个元素）
Arrays.stream(arr).count();   // 3
```

---

## Q3：如何创建一个无限流并安全使用？

**A**：无限流必须配合 `limit()` 等短路操作来限制大小：
```java
// generate 方式
Stream.generate(() -> "hello").limit(5).forEach(System.out::println);

// iterate 方式
Stream.iterate(1, n -> n * 2).limit(10).forEach(System.out::println);

// JDK 9+ 的 iterate 带终止条件
Stream.iterate(1, n -> n <= 100, n -> n * 2).forEach(System.out::println);
```
不使用 `limit()` 会导致无限循环（直到内存溢出或手动中断）。
