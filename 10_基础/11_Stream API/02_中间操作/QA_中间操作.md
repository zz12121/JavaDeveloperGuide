---
title: Stream中间操作
tags:
  - Java/Stream
  - 原理型
  - 问答
module: 11_Stream API
created: 2026-04-18
---

# Stream中间操作
## Q1：Stream 中间操作有哪些？分为哪两类？

**A**：中间操作分为两类：
**无状态操作**（每个元素独立处理）：`filter`、`map`、`flatMap`、`peek`
**有状态操作**（需依赖其他元素信息）：`distinct`、`sorted`、`limit`、`skip`
核心区别在于：有状态操作需要缓冲数据（如 sorted 需要所有元素到齐），在并行流中性能开销更大。

---

## Q2：map 和 flatMap 的区别是什么？

**A**：
- **map**：一对一转换，每个元素变成一个新元素
- **flatMap**：一对多转换，每个元素变成一个流，然后合并为一个流（扁平化）
```java
// map: List<List<Integer>> → Stream<List<Integer>>
// flatMap: List<List<Integer>> → Stream<Integer>

// 实际场景：获取所有订单中所有商品名称
orders.stream()
     .map(Order::getItems)          // List<Order> → Stream<List<Item>>
     .flatMap(List::stream)         // Stream<List<Item>> → Stream<Item>
     .map(Item::getName)            // Stream<Item> → Stream<String>
     .collect(Collectors.toList());
```

---

## Q3：distinct 是如何实现去重的？

**A**：`distinct()` 内部使用 `HashSet` 来记录已出现的元素，依赖元素的 `equals()` 和 `hashCode()` 方法判断是否重复。
对于自定义对象，如果未重写 `equals/hashCode`，distinct 不会生效（因为默认比较引用地址）。
```java
// 自定义对象去重需要重写 equals/hashCode
list.stream().distinct();
// 或者用某个唯一属性作为去重依据
list.stream()
    .collect(Collectors.toMap(User::getId, Function.identity(), (a, b) -> a))
    .values();
```

---

## Q4：limit 和 skip 在处理无限流时的行为是什么？

**A**：`limit` 是**短路操作**，在无限流中可以在有限时间内完成（取到 N 个就停）。`skip` 不是短路操作，在无限流中会导致无限等待（它需要跳过 N 个元素才能返回，但无限流永远无法完成跳过）。
```java
// 安全：limit 限制后可以处理无限流
Stream.iterate(0, n -> n + 1).limit(100).collect(Collectors.toList());

// 危险：skip 在无限流上永远无法返回
Stream.iterate(0, n -> n + 1).skip(100); // 永远不会结束！
```
