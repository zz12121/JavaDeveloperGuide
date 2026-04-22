---
title: Map 接口面试题
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Map 接口

## Q1：Map 接口有哪些常用实现类？

**A：**

| 实现类 | 底层 | 特点 |
|--------|------|------|
| HashMap | 数组+链表+红黑树 | 无序，允许 null，最常用 |
| LinkedHashMap | HashMap+双向链表 | 保持插入顺序或访问顺序 |
| TreeMap | 红黑树 | 按key排序，不允许null |
| Hashtable | 数组+链表 | 线程安全（全表锁），已过时 |
| ConcurrentHashMap | 数组+链表+红黑树 | 线程安全（高性能），多线程首选 |

---

## Q2：HashMap 和 Hashtable 的区别？

**A：**

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 不安全 | 安全（全表锁） |
| null key | 允许 1 个 | 不允许 |
| null value | 允许 | 不允许 |
| 性能 | 更快 | 较慢 |
| 推荐 | ✅ | ❌ |

---

## Q3：为什么推荐 ConcurrentHashMap 而不是 Hashtable？

**A：**
Hashtable 用 `synchronized` 给整个方法加锁（全表锁），多线程竞争激烈时性能极差。ConcurrentHashMap 在 JDK 8 中使用 **CAS + synchronized** 锁住单个桶的头节点，并发粒度更细，性能远优于 Hashtable。

---

## Q4：Map 和 Collection 是什么关系？

**A：**
Map 和 Collection 是**平级**的顶级接口，Map 不继承 Collection。但可以通过 `map.entrySet()`、`map.keySet()`、`map.values()` 获取 Collection 视图。
