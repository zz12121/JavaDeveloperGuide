---
title: Queue 接口
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Queue 接口

## Q1：Queue 的 offer / poll / peek 和 add / remove / element 有什么区别？

**A：**

| 操作 | 不抛异常（推荐） | 抛异常 |
|------|----------------|--------|
| 添加 | `offer(e)` | `add(e)` |
| 获取并移除 | `poll()` | `remove()` |
| 获取不移除 | `peek()` | `element()` |
`offer`/`poll`/`peek` 在失败时返回 null；`add`/`remove`/`element` 在失败时抛异常。推荐前者。

---

## Q2：Queue 有哪些主要实现类？

**A：**
- **LinkedList**：双向链表，同时实现 List 和 Deque
- **PriorityQueue**：基于堆的优先队列，按优先级而非 FIFO 出队
- **ArrayDeque**：基于循环数组的高性能队列

---

## Q3：Queue 的典型应用场景？

**A：**
1. **BFS 广度优先搜索**（树/图的层序遍历）
2. **生产者-消费者模式**（配合 BlockingQueue）
3. **任务调度**（按顺序处理任务）
4. **消息队列**（FIFO 消息处理）
