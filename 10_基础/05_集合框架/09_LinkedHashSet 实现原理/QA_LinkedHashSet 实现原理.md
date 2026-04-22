---
title: LinkedHashSet 实现原理
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# LinkedHashSet 实现原理

## Q1：LinkedHashSet 和 HashSet 的区别？

**A：**
- 底层：HashSet 用 HashMap，LinkedHashSet 用 **LinkedHashMap**
- 顺序：HashSet 无序，LinkedHashSet **保持插入顺序**
- 性能：LinkedHashSet 略慢（需维护双向链表）
- 内存：LinkedHashSet 略大（额外的链表指针）

---

## Q2：LinkedHashSet 是如何保持插入顺序的？

**A：**
LinkedHashSet 底层的 LinkedHashMap 在每个节点上增加了 `before` 和 `after` 两个指针，形成双向链表。每次添加新节点时，新节点被追加到链表尾部，从而维护插入顺序。遍历时沿着链表走，就是插入顺序。
