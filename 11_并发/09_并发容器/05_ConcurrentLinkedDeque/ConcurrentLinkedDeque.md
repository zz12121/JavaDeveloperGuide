---
title: ConcurrentLinkedDeque
tags:
  - Java/并发
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentLinkedDeque

## 核心结论

ConcurrentLinkedDeque 是 ConcurrentLinkedQueue 的**双端队列扩展**，支持从两端插入和删除。同样基于 CAS 无锁实现，无界非阻塞。

## 深度解析

### 结构

```
ConcurrentLinkedDeque
├── first: volatile Node（队头）
└── last: volatile Node（队尾）

Node（与 CLQ 不同）
├── prev: volatile Node（前驱指针）
├── item: volatile E（数据）
└── next: volatile Node（后继指针）
```

### API

```java
// 头部操作
offerFirst(e) / pollFirst() / peekFirst()
// 尾部操作
offerLast(e) / pollLast() / peekLast()
// 栈操作（等价于头部操作）
push(e) / pop()
// 普通队列操作
offer(e) / poll() / peek()
```

### 与 ConcurrentLinkedQueue 对比

| 维度 | CLQ | CLD |
|------|-----|-----|
| 方向 | 单端（FIFO） | 双端 |
| 节点 | next 单向 | prev + next 双向 |
| 用途 | 队列 | 队列 + 栈 + 双端 |
| 性能 | 略高（单指针） | 略低（双指针 CAS） |
| CAS 次数 | 每次操作 1~2 次 | 每次操作 2~3 次 |

### 适用场景

1. **工作窃取（Work Stealing）**：ForkJoinPool 中的任务队列
2. **栈结构**：需要 LIFO（后进先出）的并发场景
3. **双端处理**：同一队列两端都有生产者/消费者

## 关联知识点