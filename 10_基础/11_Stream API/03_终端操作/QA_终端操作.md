---
title: Stream终端操作
tags:
  - Java/Stream
  - 原理型
  - 问答
module: 11_Stream API
created: 2026-04-26
---

# Stream终端操作

## Q1：Stream 的终端操作有哪些？分为哪几类？

**A**：终端操作触发 Stream 流水线的真正执行，分为以下几类：

| 类型 | 操作 | 是否短路 |
|------|------|----------|
| 收集 | `collect` | 否 |
| 遍历 | `forEach` / `forEachOrdered` | 否 |
| 匹配 | `anyMatch` / `allMatch` / `noneMatch` | **是** |
| 查找 | `findFirst` / `findAny` | **是** |
| 聚合 | `count` / `min` / `max` | 否 |
| 归约 | `reduce` | 否 |
| 数组 | `toArray` | 否 |

> **关键点**：短路操作在找到结果后立即停止，不必处理所有元素。

---

## Q2：collect 与 Collectors 是什么关系？常见的收集器有哪些？

**A**：
- `collect()` 是 Stream 的终端方法，接收一个 `Collector` 参数
- `Collectors` 是工具类，提供了大量预定义的 `Collector` 实现

**常见收集器**：

```java
// 转集合
list.stream().collect(Collectors.toList());      // List
list.stream().collect(Collectors.toSet());       // Set
list.stream().collect(Collectors.toCollection(LinkedList::new)); // 指定集合类型

// 转 Map
list.stream().collect(Collectors.toMap(
    Person::getId,   // key
    Person::getName  // value
));

// 分组
list.stream().collect(Collectors.groupingBy(Person::getCity));

// 分区（按条件分为 true/false 两组）
list.stream().collect(Collectors.partitioningBy(p -> p.getAge() >= 18));

// 计数
list.stream().collect(Collectors.counting());

// 拼接
list.stream().collect(Collectors.joining(", "));
```

---

## Q3：reduce 和 collect 的区别是什么？何时用哪个？

**A**：

| 对比维度 | reduce | collect |
|----------|--------|---------|
| 目的 | 将流元素**归约为一个值** | 将流元素**收集到容器**中 |
| 线程安全 | 无状态，需外部同步 | 容器需线程安全，或用并发收集器 |
| 典型场景 | 求和、求最大、字符串拼接 | 分组、多值聚合、构建复杂对象 |

```java
// reduce：适合单一结果的归约
int sum = list.stream().reduce(0, Integer::sum);
Optional<Integer> max = list.stream().reduce(Integer::max);

// collect：适合多值收集
Map<String, List<Person>> byCity = list.stream()
    .collect(Collectors.groupingBy(Person::getCity));

// reduce 模拟 collect（不推荐，线程不安全）
List<String> unsafe = list.stream()
    .reduce(
        new ArrayList<String>(),
        (l, s) -> { l.add(s); return l; },
        (l1, l2) -> { l1.addAll(l2); return l1; }
    );

// ✅ collect 是正确做法
List<String> safe = list.stream().collect(Collectors.toList());
```

---

## Q4：findFirst 和 findAny 有什么区别？

**A**：
- **`findFirst`**：返回流中**第一个**元素，顺序流的行为是确定的
- **`findAny`**：返回流中**任意一个**元素，在并行流中可能比 `findFirst` 更快（不强制等待第一个）

```java
// 顺序流：两者行为一致
list.stream().findFirst();
list.stream().findAny();

// 并行流：findAny 可能返回任何元素，更快
list.parallelStream().findAny();  // 不保证返回哪个

// 如果需要确定性结果，即使在并行流中也用 findFirst
list.parallelStream().findFirst();  // 会额外同步保证返回第一个
```

> **实战建议**：非并行流用 `findFirst`，并行流且不关心顺序时用 `findAny`。

---

## Q5：forEach 和 forEachOrdered 的区别是什么？

**A**：
- `forEach`：不保证处理顺序，简单高效
- `forEachOrdered`：保证按照流的 encounter order 执行

```java
// forEach：不保证顺序（并行流中输出顺序随机）
list.parallelStream().forEach(System.out::println);

// forEachOrdered：保证顺序（即使在并行流中）
list.parallelStream().forEachOrdered(System.out::println);
```

> **注意**：`forEachOrdered` 会牺牲并行流的性能优势，因为它需要强制按顺序执行。

---

## Q6：min/max 返回 Optional 而不能直接返回元素？

**A**：`min` 和 `max` 在流为空时没有最小/最大值可返回，因此返回 `Optional`：

```java
// 空流调用 min/max 返回 Optional.empty()
Optional<Integer> min = Stream.<Integer>empty().min(Integer::compareTo);
System.out.println(min.isPresent());  // false

// 正确用法
list.stream()
    .min(Integer::compareTo)
    .orElse(Integer.MAX_VALUE);  // 提供默认值

list.stream()
    .max(Integer::compareTo)
    .orElseThrow(() -> new RuntimeException("流为空"));
```

---

## Q7：Stream 只能消费一次吗？为什么？

**A**：**是的，一个 Stream 只能被遍历一次**。因为 Stream 是单向的数据流，没有"重置"机制：

```java
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
stream.forEach(System.out::println);  // 抛 IllegalStateException

// ✅ 正确做法：如果需要多次使用，先收集
List<String> collected = list.stream().toList();  // JDK 16+
collected.forEach(System.out::println);
collected.forEach(System.out::println);  // OK
```

---

## Q8：reduce 的三参数版本和并行流有什么关系？

**A**：三参数 `reduce` 是为并行流设计的，第三参数是 **combiner**（合并器）：

```java
// 单参数 reduce（无初始值）
Optional<T> result = stream.reduce((a, b) -> ...);

// 两参数 reduce（有初始值）
T result = stream.reduce(identity, (a, b) -> ...);

// 三参数 reduce（用于并行流）
T result = stream.parallelStream()
    .reduce(
        identity,           // 初始值（累加器初始值）
        accumulator,        // 累加器：各线程内部合并
        combiner            // 合并器：各线程结果合并
    );
```

```java
// 示例：并行求和
int sum = IntStream.range(1, 1000)
    .parallel()
    .reduce(
        0,                  // 初始值
        Integer::sum,       // 累加器
        Integer::sum        // 合并器
    );
```

> **注意**：累加器和合并器的操作必须是**结合律**的（a op b op c = (a op b) op c = a op (b op c)）。
