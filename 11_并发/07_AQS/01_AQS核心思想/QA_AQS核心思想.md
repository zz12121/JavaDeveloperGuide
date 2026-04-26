---
title: AQS核心思想
tags:
  - Java/并发
  - 问答
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# AQS核心思想（状态state + FIFO队列 + CAS + Park/Unpark）

## Q1：什么是 AQS？它的核心设计是什么？

**A**：AQS（AbstractQueuedSynchronizer）是 JUC 并发框架的核心抽象类，设计包含四大要素：

1. **state（volatile int）**：同步状态，由子类定义语义（锁计数、剩余计数、许可数等）
2. **FIFO 队列（CLH 变体）**：双向链表，管理等待获取锁的线程
3. **CAS（Unsafe）**：原子修改 state，保证线程安全
4. **LockSupport（park/unpark）**：阻塞和唤醒线程

核心模式：**模板方法模式**——AQS 定义 acquire/release 流程，子类只需实现 tryAcquire/tryRelease。

---

## Q2：AQS 的 CLH 队列是怎么工作的？

**A**：CLH 队列是 AQS 的等待队列，每个等待线程封装为一个 Node：

```
head(哨兵) ← Node(thread=B, ws=SIGNAL) ← Node(thread=C) ← tail
```

- **入队**：获取锁失败的线程封装为 Node，CAS 挂到 tail 上
- **自旋**：队列中的节点检查前驱是否为 head，是则再 tryAcquire
- **park**：前驱不是 head 或 tryAcquire 失败，则 park 阻塞
- **unpark**：释放锁的线程 unpark 后继节点，后继节点被唤醒后自旋

---

## Q3：哪些并发工具基于 AQS 实现？

**A**：

| 工具 | 模式 | state 语义 |
|------|------|-----------|
| ReentrantLock | 独占 | 0=未锁, >0=重入次数 |
| ReentrantReadWriteLock | 独占+共享 | 高16位=读锁, 低16位=写锁 |
| CountDownLatch | 共享 | 剩余计数 |
| Semaphore | 共享 | 可用许可数 |
| CyclicBarrier | (基于 ReentrantLock) | — |
| FutureTask | 独占 | 任务状态 |
