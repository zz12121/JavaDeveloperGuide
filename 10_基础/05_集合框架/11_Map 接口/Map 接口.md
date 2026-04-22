---
title: Map 接口
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# Map 接口（HashMap / LinkedHashMap / TreeMap / Hashtable / ConcurrentHashMap）

## Map 接口特点

`Map` 是**独立于 Collection** 的顶级接口，存储键值对（Key-Value）：

| 特点 | 说明 |
|------|------|
| 键值对存储 | 每个元素是一个 `Entry<K, V>` |
| Key 唯一 | 不允许重复的 key |
| Value 可重复 | 多个 key 可以映射相同的 value |
| 一个 null key | HashMap/LinkedHashMap/Hashtable 各有不同 |

## 核心方法

```java
V put(K key, V value);           // 添加键值对
V get(Object key);               // 获取 value
boolean containsKey(Object key); // 是否包含 key
boolean containsValue(Object v); // 是否包含 value
Set<K> keySet();                 // 所有 key
Collection<V> values();          // 所有 value
Set<Map.Entry<K,V>> entrySet();  // 所有键值对
int size();
V remove(Object key);
```

## 五个实现类对比

| 维度 | HashMap | LinkedHashMap | TreeMap | Hashtable | ConcurrentHashMap |
|------|---------|---------------|---------|-----------|-------------------|
| 底层 | 数组 + 链表 + 红黑树 | HashMap + 双向链表 | 红黑树 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| key 顺序 | 无序 | 插入序/访问序 | 排序 | 无序 | 无序 |
| null key | ✅ 允许 1 个 | ✅ 允许 1 个 | ❌ 不允许 | ❌ 不允许 | ❌ 不允许 |
| null value | ✅ 允许多个 | ✅ 允许多个 | ❌ 不允许 | ❌ 不允许 | ❌ 不允许 |
| 线程安全 | ❌ | ❌ | ❌ | ✅（全表锁） | ✅（分段锁/CAS） |
| 性能 | O(1) | O(1) | O(log n) | O(1) | O(1) |
| JDK 版本 | 1.2 | 1.4 | 1.2 | 1.0 | 1.5 |

## HashMap

最常用的 Map 实现，JDK 8 采用数组 + 链表 + 红黑树结构。

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.get("a");  // 1
```

## LinkedHashMap

在 HashMap 基础上维护双向链表：

```java
// accessOrder = false → 维护插入顺序（默认）
// accessOrder = true  → 维护访问顺序（LRU）
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true);
```

## TreeMap

基于红黑树的有序 Map：

```java
// 自然排序
Map<String, Integer> treeMap = new TreeMap<>();

// 自定义排序
Map<String, Integer> treeMap = new TreeMap<>(Comparator.reverseOrder());
```

## Hashtable

古老的线程安全 Map，已不推荐使用：

```java
// 所有方法都加了 synchronized（全表锁）
public synchronized V put(K key, V value) { ... }
public synchronized V get(Object key) { ... }
```

**缺点**：锁粒度大，性能差；不允许 null key/value。

## ConcurrentHashMap

高性能线程安全 Map：

- JDK 7：**分段锁**（Segment 数组，每个 Segment 独立加锁）
- JDK 8：**CAS + synchronized**（锁粒度细化到每个桶的头节点）

## 选择指南

| 需求 | 选择 |
|------|------|
| 默认选择 | **HashMap** |
| 需要保持插入/访问顺序 | **LinkedHashMap** |
| 需要排序 | **TreeMap** |
| 多线程环境 | **ConcurrentHashMap** |
| 不用 Hashtable | ❌ |

## 关联知识点
