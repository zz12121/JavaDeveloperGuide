---
title: LinkedList 底层实现
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# LinkedList 底层实现

## Q1：LinkedList 的底层实现原理？

**A：**
LinkedList 底层是**双向链表**，通过 `Node` 节点存储元素，每个节点包含 `item`（数据）、`prev`（前驱）、`next`（后继）三个字段。同时实现了 `List` 和 `Deque` 接口。

---

## Q2：LinkedList 为什么随机访问慢？

**A：**
LinkedList 没有实现 `RandomAccess` 接口，`get(index)` 需要从头或从尾遍历链表到指定位置，时间复杂度 **O(n)**。虽然源码做了折半优化（索引在前半部分从头遍历，后半部分从尾遍历），但仍然是线性时间。

---

## Q3：LinkedList 的插入删除一定是 O(1) 吗？

**A：**
不完全是。**头尾插入删除是 O(1)**，但中间位置插入删除是 **O(n)**，因为需要先遍历找到目标位置（`node(index)` 方法）。找到位置后，修改指针的操作本身是 O(1)。

---

## Q4：LinkedList 可以当栈和队列用吗？

**A：**
可以。LinkedList 实现了 `Deque` 接口：
- 当栈：`push()` / `pop()` / `peek()`（操作头部）
- 当队列：`offer()` / `poll()` / `peek()`（FIFO）
- 当双端队列：`addFirst()` / `addLast()` / `removeFirst()` / `removeLast()`
