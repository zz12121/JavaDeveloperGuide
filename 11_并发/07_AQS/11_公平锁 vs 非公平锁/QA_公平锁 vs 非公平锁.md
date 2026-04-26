---
title: 公平锁 vs 非公平锁
tags:
  - Java/并发
  - 问答
  - 对比型
module: 07_AQS
created: 2026-04-18
---

# 公平锁 vs 非公平锁（公平锁检查队列，非公平锁直接CAS抢锁）

## Q1：公平锁和非公平锁的核心区别是什么？

**A**：

- **非公平锁**：`tryAcquire` 时直接 CAS 抢锁，不管队列有没有等待者
- **公平锁**：`tryAcquire` 时先调用 `hasQueuedPredecessors()` 检查队列，有等待者则排队

```java
// 非公平
if (compareAndSetState(0, 1)) return true;

// 公平
if (!hasQueuedPredecessors() && compareAndSetState(0, 1)) return true;
```

---

## Q2：为什么默认用非公平锁？

**A**：非公平锁吞吐量更高：

1. **减少线程切换**：新线程 CAS 抢锁可能成功，不需要 park/unpark
2. **唤醒延迟**：被唤醒的线程从 park 到 tryAcquire 有时间差，这期间新线程可以直接获取
3. **实际公平性影响小**：大多数场景下线程等待时间差距不大

`synchronized` 和 `ReentrantLock` 默认都是非公平锁。

---

## Q3：什么场景需要用公平锁？

**A**：当**线程饥饿**问题不可接受时：

- 按顺序处理任务的队列（如打印队列）
- 公平性比吞吐量更重要的业务
- 需要严格 FIFO 保证的系统

```java
// 打印队列：严格按提交顺序打印
private final Lock printLock = new ReentrantLock(true); // 公平锁
```

但大多数场景下非公平锁更好，因为公平锁每次 tryAcquire 都要检查队列，增加了开销。

