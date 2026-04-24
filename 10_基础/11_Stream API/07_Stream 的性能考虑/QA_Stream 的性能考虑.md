---
title: Stream的性能考虑
tags:
  - Java/Stream
  - 场景型
  - 问答
module: 11_Stream API
created: 2026-04-18
---

# Stream的性能考虑
## Q1：Stream 和传统 for 循环哪个更快？

**A**：**大多数场景下差距不大，现代 JDK 中已接近**。

- 小数据集（<1000）：for 循环略快，因为 Stream 有初始化开销（Spliterator、lambda 对象创建）
- 大数据集（>10000）：性能接近，JIT 优化后 Stream 可能与 for 循环持平
- 并行流：大数据集 + CPU 密集型操作下可能更快
**建议**：非热点路径优先考虑 Stream 的可读性，只有在性能测试证明瓶颈后再考虑 for 循环。

---

## Q2：如何优化 Stream 的性能？

**A**：核心优化点：
1. **使用特化流避免装箱**：`mapToInt`/`mapToLong`/`mapToDouble`
2. **filter 前置**：先过滤减少后续操作的数据量
3. **方法引用替代 lambda**：`String::length` 比 `s -> s.length()` 更利于 JIT 内联
4. **用 collect 替代 forEach + 外部变量**：collect 内部更高效且线程安全
5. **大数据集才考虑并行流**：小数据集并行流反而更慢
```java
// 典型优化
list.stream()
    .filter(Objects::nonNull)          // 1. 先过滤
    .mapToInt(String::length)           // 2. 特化流避免装箱
    .summaryStatistics();               // 3. 一次性获取所有统计
```

---

## Q3：为什么装箱拆箱对 Stream 性能影响很大？

**A**：
Java 泛型不支持基本类型，`Stream<T>` 中的 T 必须是对象类型。当对 `int` 数据使用 `Stream<Integer>` 时：
- 每个 int 自动装箱为 Integer（分配堆内存）
- 每个 Integer 操作后又拆箱为 int
这种装箱/拆箱在大量数据时产生**大量临时对象**，导致：
- GC 压力增大
- CPU 缓存命中率下降
- 内存占用增加
```java
// Stream<Integer>：每次操作都有装箱
Stream<Integer> s = list.stream().map(i -> i * 2);

// IntStream：无装箱
IntStream s = list.stream().mapToInt(i -> i).map(i -> i * 2);
```

---

## Q4：什么场景下不建议使用 Stream？

**A**：
1. **极致性能的热点路径**：每秒调用百万次的代码段
2. **需要 break/continue 的复杂控制流**：Stream 不支持非结构化跳转
3. **需要修改外部可变状态**：违背 Stream 的函数式设计
4. **循环中需要抛出受检异常**：lambda 不支持 throws 声明
5. **需要精确索引控制**：Stream 没有索引概念
```java
// 需要同时处理当前元素和上一个元素 → 不适合 Stream
for (int i = 1; i < list.size(); i++) {
    compare(list.get(i - 1), list.get(i)); // 需要相邻元素比较
}
```
