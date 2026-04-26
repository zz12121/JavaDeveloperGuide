---
title: AQS两种模式
tags:
  - Java/并发
  - 问答
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# AQS两种模式（独占模式ReentrantLock/共享模式CountDownLatch/Semaphore）

## Q1：AQS 的独占模式和共享模式有什么区别？

**A**：

| 维度 | 独占模式 | 共享模式 |
|------|---------|---------|
| 持有者 | 同时只有一个线程 | 可以多个线程同时持有 |
| 获取方法 | `acquire()` / `tryAcquire()` | `acquireShared()` / `tryAcquireShared()` |
| 释放方法 | `release()` / `tryRelease()` | `releaseShared()` / `tryReleaseShared()` |
| 唤醒传播 | 只唤醒一个后继 | 向后传播，可能唤醒多个 |
| 典型实现 | ReentrantLock, WriteLock | Semaphore, CountDownLatch, ReadLock |

核心区别在于释放时：独占模式只 unpark 一个后继节点，共享模式会**传播唤醒**（如果后继也是共享模式）。

---

## Q2：共享模式的唤醒传播是怎么工作的？

**A**：`setHeadAndPropagate` 是共享模式独有的机制：

```java
private void setHeadAndPropagate(Node node, int propagate) {
    setHead(node); // 设置为新的 head
    if (propagate > 0 || node.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared(); // 继续唤醒后继
    }
}
```

当共享锁释放后，如果还有许可，会继续唤醒队列中下一个共享模式的节点，形成**链式唤醒传播**。

---

## Q3：哪些并发工具用了独占模式，哪些用了共享模式？

**A**：

- **独占**：`ReentrantLock`、`ReentrantReadWriteLock.WriteLock`
- **共享**：`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock.ReadLock`
- **两者兼有**：`ReentrantReadWriteLock`（读锁共享，写锁独占）
