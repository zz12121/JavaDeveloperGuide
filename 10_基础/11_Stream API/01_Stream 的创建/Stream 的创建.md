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

---

## 关联知识点
