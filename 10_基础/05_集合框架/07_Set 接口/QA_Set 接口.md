---
title: Set 接口
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Set 接口

## Q1：HashSet、LinkedHashSet、TreeSet 的区别？

**A：**

| 维度 | HashSet | LinkedHashSet | TreeSet |
|------|---------|---------------|---------|
| 底层 | HashMap | LinkedHashMap | TreeMap |
| 顺序 | 无序 | 插入顺序 | 排序 |
| null | 允许 | 允许 | 不允许 |
| 性能 | O(1) | O(1) | O(log n) |

---

## Q2：Set 是如何保证元素不重复的？

**A：**
- **HashSet / LinkedHashSet**：调用元素的 `hashCode()` 定位桶，再用 `equals()` 判断是否重复
- **TreeSet**：调用元素的 `compareTo()`（或 Comparator）判断是否重复（返回 0 表示重复）

---

## Q3：TreeSet 为什么不允许 null？

**A：**
TreeSet 底层是 TreeMap，使用 `compareTo()` 比较元素，`null.compareTo(...)` 会抛 `NullPointerException`。
