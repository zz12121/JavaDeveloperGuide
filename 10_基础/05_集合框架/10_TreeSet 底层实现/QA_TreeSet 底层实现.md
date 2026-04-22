---
title: TreeSet 底层实现面试题
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# TreeSet 底层实现

## Q1：TreeSet 的底层实现是什么？

**A：**
TreeSet 底层是 **TreeMap**（红黑树）。元素作为 TreeMap 的 key 存储，value 是固定空对象。所有操作的时间复杂度为 O(log n)。

---

## Q2：TreeSet 是如何保证元素有序且不重复的？

**A：**
- **有序**：红黑树是自平衡二叉搜索树，中序遍历就是排序结果
- **去重**：通过 `compareTo()` 判断，返回 0 表示重复元素

---

## Q3：TreeSet 的去重和 HashSet 有什么区别？

**A：**
- HashSet 通过 `hashCode()` + `equals()` 去重
- TreeSet 通过 `compareTo()` / `Comparator` 去重
- 如果 `compareTo()` 返回 0 但 `equals()` 返回 false，TreeSet 认为重复而 HashSet 不认为重复，导致两者行为不一致

---

## Q4：TreeSet 为什么不允许 null？

**A：**
TreeSet 使用 `compareTo()` 比较元素，`null.compareTo(...)` 会抛 `NullPointerException`。
