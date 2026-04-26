---
title: Stream的创建
tags:
  - Java/Stream
  - 原理型
module: 11_Stream API
created: 2026-04-18
---

## 核心结论

Stream 的创建主要有三种方式：**Collection 接口的 stream()/parallelStream()**、**Arrays.stream()**、**Stream.of()**，此外还有 **Stream.generate()/Stream.iterate()** 用于创建无限流。

---

## 深度解析

### 1. Collection → Stream

最常用的方式，任何 Collection 实现类均可调用：

```java
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream = list.stream();           // 顺序流
Stream<String> parallel = list.parallelStream(); // 并行流
```

### 2. Arrays → Stream

通过 `Arrays.stream()` 将数组转为 Stream：

```java
int[] arr = {1, 2, 3};
IntStream stream = Arrays.stream(arr);

String[] strArr = {"a", "b", "c"};
Stream<String> stream2 = Arrays.stream(strArr);
```

注意基本类型数组返回的是特化流（`IntStream`、`LongStream`、`DoubleStream`）。

### 3. Stream.of()

通过工厂方法直接创建：

```java
Stream<String> s1 = Stream.of("a", "b", "c");
Stream<Integer> s2 = Stream.of(1, 2, 3);

// 注意：基本类型数组会被当作单个元素
int[] arr = {1, 2, 3};
Stream<int[]> s3 = Stream.of(arr); // Stream<int[]>，不是 IntStream！
```

### 4. 无限流

```java
// generate：无参 Supplier，不断生成相同或随机值
Stream<Double> randoms = Stream.generate(Math::random).limit(10);

// iterate：种子 + UnaryOperator，逐个生成
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1).limit(100);

// JDK 9+ iterate 支持 Predicate 终止条件
Stream<Integer> range = Stream.iterate(0, n -> n < 100, n -> n + 1);
```

### 5. 其他创建方式

```java
// 字符串
IntStream chars = "hello".chars();  // 字符的 IntStream

// 范围（IntStream/LongStream 专属）
IntStream range1 = IntStream.range(1, 5);     // [1, 2, 3, 4]
IntStream range2 = IntStream.rangeClosed(1, 5); // [1, 2, 3, 4, 5]

// 空流
Stream<String> empty = Stream.empty();

// 合并
Stream<String> concat = Stream.concat(Stream.of("a"), Stream.of("b"));
```

### 6. 创建方式对比

| 方式 | 特点 | 适用场景 |
|------|------|----------|
| `collection.stream()` | 最常用，适用于任何集合 | 集合数据处理 |
| `Arrays.stream()` | 支持基本类型特化流 | 数组处理 |
| `Stream.of()` | 简洁，但不支持基本类型数组拆箱 | 少量元素 |
| `Stream.generate()` | 无限流，需 limit 限制 | 生成随机数/常量序列 |
| `Stream.iterate()` | 无限流，递推生成 | 等差/递推序列 |
| `IntStream.range()` | 范围数值流 | 数值循环替代 |

### 7. 特化流详解 (IntStream / LongStream / DoubleStream)

Java 提供了三种数值特化流，避免装箱/拆箱开销：

#### 创建特化流

```java
// IntStream
IntStream intStream = IntStream.of(1, 2, 3);
IntStream rangeStream = IntStream.range(1, 100);        // 左闭右开 [1, 100)
IntStream closedStream = IntStream.rangeClosed(1, 100);  // 左闭右闭 [1, 100]

// LongStream
LongStream longStream = LongStream.rangeClosed(1, 1_000_000);

// DoubleStream
DoubleStream doubleStream = DoubleStream.generate(Math::random);
```

#### IntStream 常用方法

| 方法 | 说明 | 示例 |
|------|------|------|
| `sum()` | 求和 | `IntStream.of(1,2,3).sum()` → 6 |
| `average()` | 求平均值，返回 OptionalDouble | `IntStream.of(1,2,3).average()` → OptionalDouble[2.0] |
| `min()` / `max()` | 最小/最大值，返回 OptionalInt | `IntStream.of(1,2,3).max()` → OptionalInt[3] |
| `range(a, b)` | 左闭右开范围 | `IntStream.range(1, 5)` → [1,2,3,4] |
| `rangeClosed(a, b)` | 左闭右闭范围 | `IntStream.rangeClosed(1, 5)` → [1,2,3,4,5] |
| `boxed()` | 转回包装类型 Stream | `intStream.boxed()` → Stream<Integer> |
| `mapToObj()` | 映射为对象流 | `intStream.mapToObj(String::valueOf)` |
| `summaryStatistics()` | 一次性获取所有统计 | 返回 IntSummaryStatistics |
| `takeWhile()` | JDK 9+，条件满足时取元素 | `IntStream.range(1,10).takeWhile(x -> x < 5)` |
| `dropWhile()` | JDK 9+，跳过满足条件的元素 | `IntStream.range(1,10).dropWhile(x -> x < 5)` |

#### 统计汇总

```java
IntSummaryStatistics stats = IntStream.rangeClosed(1, 100)
    .summaryStatistics();

System.out.println("Count: " + stats.getCount());      // 100
System.out.println("Sum: " + stats.getSum());          // 5050
System.out.println("Min: " + stats.getMin());          // 1
System.out.println("Max: " + stats.getMax());          // 100
System.out.println("Average: " + stats.getAverage());  // 50.5
```

#### mapToXxx 系列（与其他 Stream 互转）

```java
// Stream<T> → 特化流
int sum = list.stream().mapToInt(Integer::intValue).sum();
long max = list.stream().mapToLong(Long::longValue).max().orElse(0);

// 特化流 → Stream<T>
Stream<Integer> boxed = intStream.boxed();
Stream<String> asStrings = intStream.mapToObj(String::valueOf);

// IntStream → DoubleStream
DoubleStream doubles = intStream.asDoubleStream();
```

#### 生成斐波那契数列示例

```java
// 使用 iterate 生成斐波那契数列
Stream.iterate(new long[]{1, 1}, f -> new long[]{f[1], f[0] + f[1]})
    .limit(10)
    .map(f -> f[0])
    .forEach(System.out::println);
```

---

## 关联知识点
