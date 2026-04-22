---
title: PECS 原则
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# PECS 原则

## Q1：什么是 PECS 原则？

**A：**
PECS 是 **Producer Extends, Consumer Super** 的缩写，指导泛型通配符的使用：

- **Producer（生产者，读数据）→ 使用 `? extends T`**：集合对你来说是数据来源，只读取
- **Consumer（消费者，写数据）→ 使用 `? super T`**：集合对你来说是数据目标，只写入

```java
// 典型例子：复制集合
<T> void copy(
    List<? super T> dest,    // 消费者：往里写，用 super
    List<? extends T> src    // 生产者：从里读，用 extends
) {
    for (T item : src) dest.add(item);
}
```

---

## Q2：以下代码用 PECS 原则应该怎么设计？

> 场景：写一个方法，从 `source` 集合中找到最大元素，添加到 `sink` 集合中。

**A：**
```java
public static <T extends Comparable<? super T>> void addMax(
    List<? extends T> source,   // 生产者：读取元素
    List<? super T> sink        // 消费者：写入最大值
) {
    if (source.isEmpty()) return;
    T max = Collections.max(source);  // 从 source 读
    sink.add(max);                    // 写入 sink
}

// 使用
List<Integer> ints = Arrays.asList(3, 1, 4, 1, 5);
List<Number> numbers = new ArrayList<>();
addMax(ints, numbers);  // ✅ Integer extends T，Number super T
```

---

## Q3：如果既要读又要写，用什么通配符？

**A：**
**不用通配符，直接用具体类型参数 `T`**：

```java
// 既要读又要写
public <T> void swap(List<T> list, int i, int j) {
    T temp = list.get(i);   // 读
    list.set(i, list.get(j));
    list.set(j, temp);      // 写
}
```

只有"单向操作"时才能用通配符；双向操作必须用具体类型 `T`。

---

## Q4：`Collections.sort` 方法签名 `sort(List<T>, Comparator<? super T>)` 为什么 Comparator 用 `? super T`？

**A：**
因为 `Comparator` 是数据的**消费者**——它消费（接收）两个 T 类型的元素进行比较，不产出数据。按 PECS 原则，消费者用 `? super T`。

好处：`List<Integer>` 可以使用 `Comparator<Number>` 来排序——只要比较器能处理 `Number` 及其父类，就能比较 `Integer`：

```java
List<Integer> list = Arrays.asList(3, 1, 2);
Comparator<Number> comp = Comparator.comparingDouble(Number::doubleValue);
list.sort(comp);  // ✅ 因为 Comparator<? super Integer> 接受 Comparator<Number>
```
