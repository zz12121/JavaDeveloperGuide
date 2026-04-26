---
title: LinkedBlockingDeque
tags:
  - Java/并发
  - 原理型
  - 问答
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedBlockingDeque

## Q1：LinkedBlockingDeque 和 LinkedBlockingQueue 有什么区别？

**A**：

- **LBQ**：单向链表，只能从尾部插入、头部删除（FIFO）
- **LBD**：双向链表，支持从两端插入和删除

LBD 使用单锁（因为双端操作需要统一），LBQ 使用双锁（单端操作可以分离）。

---

## Q2：LinkedBlockingDeque 的典型应用场景？

**A**：

1. **用户层工作窃取**：生产者从队列一端放任务，多个消费者从另一端取任务（类似 ForkJoinPool 的工作窃取思想，但 ForkJoinPool 内部使用 WorkQueue 而非 LBD）
2. **双端处理**：生产者头部放、消费者尾部取
3. **并发栈**：push/pop 操作

> **注意**：ForkJoinPool 内部使用自己实现的 `WorkQueue`（数组 + CAS），不是 LinkedBlockingDeque。



> **代码示例：LinkedBlockingDeque 双端操作**

```java
LinkedBlockingDeque<String> deque = new LinkedBlockingDeque<>(10);

// 双端插入
deque.addFirst("head");   // 头部插入
deque.addLast("tail");    // 尾部插入

// 双端取出
String first = deque.takeFirst();  // 阻塞取头部
String last = deque.takeLast();    // 阻塞取尾部

// 作为栈使用
deque.push("A");  // 等价于 addFirst
deque.pop();      // 等价于 removeFirst

// 作为工作窃取队列：生产者放一端，消费者/窃取者取另一端
```
