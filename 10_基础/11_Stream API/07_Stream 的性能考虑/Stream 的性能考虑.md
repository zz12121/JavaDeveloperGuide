---
title: Stream的性能考虑
tags:
  - Java/Stream
  - 场景型
module: 11_Stream API
created: 2026-04-18
---

## 核心结论

Stream 在大多数场景下性能与 for 循环接近，但在某些场景下有明显差距。关键原则：**先 filter 再 map、避免装箱拆箱、大数据集才考虑并行流、注意有状态操作的开销**。

---

## 深度解析

### 1. Stream vs 传统 for 循环性能

| 维度 | Stream | for 循环 |
|------|--------|----------|
| 小数据集 | 略慢（初始化开销） | 更快 |
| 大数据集 | 接近 for 循环 | 基准线 |
| 可读性 | 更好（声明式） | 较差（命令式） |
| JIT 优化 | JDK 不断优化中 | 非常成熟 |

> **结论**：在现代 JDK（11+）中，Stream 性能差距已大幅缩小，大多数情况下**可读性优先**。

### 2. 核心性能优化点

#### ① 使用原始类型特化流避免装箱

```java
// ❌ 慢：每次操作都有 Integer 装箱/拆箱
int sum = list.stream()
    .map(i -> i * 2)
    .reduce(0, Integer::sum);

// ✅ 快：IntStream 无装箱开销
int sum = list.stream()
    .mapToInt(Integer::intValue)
    .map(i -> i * 2)
    .sum();
```

#### ② filter 放在 map 之前

```java
// ❌ 先 map 所有元素，再过滤
list.stream()
    .map(this::expensiveTransform) // 所有元素都执行
    .filter(s -> s != null)        // 然后过滤
    .collect(Collectors.toList());

// ✅ 先过滤，减少 map 调用次数
list.stream()
    .filter(Objects::nonNull)       // 先过滤
    .map(this::expensiveTransform)  // 只对有效元素执行
    .collect(Collectors.toList());
```

#### ③ 有状态操作的性能陷阱

```java
// distinct 需要维护 HashSet，sorted 需要完整缓冲
// 在 Stream 中频繁使用有状态操作会显著降低性能

// sorted + limit 的陷阱：先排全量，再截取
list.stream().sorted().limit(5); // 排序全部元素再取前5

// 如果只需要前5个最大值，用更高效的方式：
// PriorityQueue 或 Collections.nCopies 模拟
```

#### ④ 避免在 Stream 中修改外部状态

```java
// ❌ 修改外部变量（副作用，且并行流中不安全）
Map<String, List<Item>> map = new HashMap<>();
list.stream().forEach(item -> {
    map.computeIfAbsent(item.getType(), k -> new ArrayList<>()).add(item);
});

// ✅ 使用 collect（线程安全，声明式）
Map<String, List<Item>> map = list.stream()
    .collect(Collectors.groupingBy(Item::getType));
```

### 3. Stream 性能优化清单

| 优化项 | 说明 |
|--------|------|
| 用 IntStream/LongStream | 避免装箱拆箱 |
| filter 前置 | 减少后续操作的数据量 |
| 避免频繁 boxed/unboxed | 保持流类型一致 |
| 优先使用 collect | 比手动 forEach + 外部变量更高效 |
| 谨慎使用并行流 | 小数据集反而更慢 |
| 避免复杂 lambda | 提取为方法引用，利于 JIT 内联 |
| 注意 sorted 的 O(n log n) | 有排序需求时考虑是否真的需要全排序 |

### 4. 何时不用 Stream

- 极致性能要求的**热点路径**（每秒调用百万次）
- 需要**精确控制循环**（如 break + 标签、continue）
- 需要**修改循环变量索引**（如步长遍历）
- 需要**循环中抛出受检异常**（lambda 不支持）

---

## 关联知识点

