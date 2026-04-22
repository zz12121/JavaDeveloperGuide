---
title: Deque 接口面试题
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Deque 接口

## Q1：Deque 是什么？和 Queue 有什么关系？

**A：**
Deque（Double-Ended Queue）是双端队列，继承自 Queue。Queue 只能在尾部添加、头部删除（FIFO），Deque 支持在**两端**进行添加和删除操作。Deque 同时兼具**队列**和**栈**的功能。

---

## Q2：为什么推荐 ArrayDeque 替代 Stack？

**A：**
1. `Stack` 继承 `Vector`，所有方法 `synchronized`，性能差
2. `Stack` 已被标记为过时
3. `ArrayDeque` 底层是循环数组，CPU 缓存友好，没有对象创建开销，**性能远优于 LinkedList 和 Stack**
```java
// ❌ 过时
Stack<String> stack = new Stack<>();

// ✅ 推荐
Deque<String> stack = new ArrayDeque<>();
```

---

## Q3：ArrayDeque 和 LinkedList 作为 Deque 的区别？

**A：**

| 维度 | ArrayDeque | LinkedList |
|------|-----------|------------|
| 底层 | 循环数组 | 双向链表 |
| 内存 | 连续，缓存友好 | 分散，额外指针 |
| 对象创建 | 无（数组赋值） | 有（new Node） |
| 性能 | **更快** | 较慢 |
| 实现 | 仅 Deque | List + Deque |
绝大多数场景推荐 ArrayDeque。只有需要同时按索引访问时才考虑 LinkedList。
