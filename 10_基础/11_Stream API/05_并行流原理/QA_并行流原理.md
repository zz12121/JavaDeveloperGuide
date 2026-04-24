---
title: 并行流原理
tags:
  - Java/Stream
  - 原理型
  - 问答
module: 11_Stream API
created: 2026-04-18
---

# 并行流原理
## Q1：并行流的底层实现原理是什么？

**A**：并行流基于 **ForkJoinPool** 实现：
1. 使用 `Spliterator` 将数据源递归拆分为更小的子任务
2. 子任务提交到 ForkJoinPool 的工作窃取队列
3. 各线程并行处理子任务
4. 子任务结果通过 combiner 函数逐层合并为最终结果
默认使用 `ForkJoinPool.commonPool()`，线程数为 CPU 核心数 - 1。

---

## Q2：什么场景下适合使用并行流？什么场景不适合？

**A**：
**适合**：大数据集（>1万）、无状态操作、计算密集型、ArrayList 等可精确拆分的数据源
**不适合**：
- 小数据集（线程调度开销大于并行收益）
- 有状态操作（sorted、distinct 需要全局协调）
- IO 密集型任务
- 需要保证处理顺序
- LinkedList 等难以拆分的数据源
> **经验法则**：当数据量小或操作简单时，顺序流通常更快。并行流不是"免费的性能提升"。

---

## Q3：并行流中使用 forEach 操作 ArrayList 会有什么问题？

**A**：`ArrayList` 不是线程安全的，并行流中的多个线程同时调用 `add` 会导致：
1. **数据丢失** — 并发修改内部数组导致覆盖
2. **ArrayIndexOutOfBoundsException** — size 和数组实际大小不一致
3. **数据重复** — 并发读取相同的 size 值
```java
// ❌ 危险：非线程安全的 ArrayList + 并行流
List<Integer> list = new ArrayList<>();
IntStream.range(0, 10000).parallel().forEach(list::add);
System.out.println(list.size()); // 不一定是 10000！

// ✅ 安全：使用 collect
List<Integer> list = IntStream.range(0, 10000)
    .parallel()
    .boxed()
    .collect(Collectors.toList());
```

---

## Q4：如何为并行流指定自定义线程池？

**A**：
```java
// 方式一：通过自定义 ForkJoinPool 提交任务
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    list.parallelStream().map(...).collect(...)
).get();

// 方式二：全局设置（影响所有并行流，不推荐）
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "4");
```

推荐方式一，避免全局设置影响其他并行流。
