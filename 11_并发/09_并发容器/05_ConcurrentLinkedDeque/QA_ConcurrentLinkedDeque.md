---
title: ConcurrentLinkedDeque
tags:
  - Java/并发
  - 问答
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentLinkedDeque

## Q1：ConcurrentLinkedDeque 和 ConcurrentLinkedQueue 有什么区别？

**A**：

- **CLQ**：单向链表，只能从尾部插入、头部删除（FIFO）
- **CLD**：双向链表，支持从两端插入和删除

CLD 的 Node 多了一个 `prev` 指针，CAS 操作更复杂（需要维护双向链接），性能略低于 CLQ。

---

## Q2：ConcurrentLinkedDeque 的典型应用场景？

**A**：

1. **ForkJoinPool 工作窃取**：每个线程有自己的双端队列，从头部取任务，其他线程从尾部"窃取"
2. **并发栈**：push/pop 操作（等价于 offerFirst/pollFirst）
3. **双端处理**：生产者从一头放，消费者从另一头取

---
```java
// ConcurrentLinkedDeque — 非阻塞双端队列
ConcurrentLinkedDeque<String> deque = new ConcurrentLinkedDeque<>();

deque.addFirst("A");   // 头部插入
deque.addLast("B");    // 尾部插入
deque.addFirst("C");   // 头部插入 → [C, A, B]

String first = deque.pollFirst();  // "C"（FIFO 模式）
String last = deque.pollLast();    // "B"（栈模式）

// 所有操作通过 CAS 实现无锁并发
```


