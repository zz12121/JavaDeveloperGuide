---
title: ArrayBlockingQueue
tags:
  - Java/并发
  - 问答
  - 源码型
module: 10_阻塞队列
created: 2026-04-18
---

# ArrayBlockingQueue

## Q1：ArrayBlockingQueue 的实现原理？

**A**：

- **环形数组**存储元素，takeIndex/putIndex 环形递增
- **单 ReentrantLock** 保护所有操作
- **两个 Condition**：notEmpty（take 等待）/ notFull（put 等待）
- put 时队列满 → notFull.await()；take 时队列空 → notEmpty.await()
- 入队/出队后分别 signal 对应条件

---

## Q2：ArrayBlockingQueue 为什么用单锁而不是双锁？

**A**：因为数组是共享的连续内存，putIndex 和 takeIndex 都在同一数组上操作，必须用同一把锁保证原子性。

LinkedBlockingQueue 可以用双锁（putLock/takeLock），因为链表的 head 和 tail 是独立的 Node 引用，可以分开加锁。

---

## Q3：ArrayBlockingQueue 的公平模式是什么？

**A**：构造参数 `new ArrayBlockingQueue(10, true)` 启用公平模式。

公平模式下，等待 put/take 的线程按 FIFO 顺序被唤醒，减少线程饥饿。默认非公平模式吞吐量更高。

---
```java
// ArrayBlockingQueue：环形数组 + 单锁
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10, true);  // 公平模式

queue.put("a");  // 队列满 → notFull.await()
queue.put("b");  // 队列满 → notFull.await()
queue.take();    // 取出 "a" → notFull.signal()
queue.take();    // 取出 "b"

// 内部结构：Object[] items + putIndex + takeIndex
// put 时：items[putIndex] = e; putIndex = (putIndex+1) % items.length;
// take 时：e = items[takeIndex]; takeIndex = (takeIndex+1) % items.length;
```

