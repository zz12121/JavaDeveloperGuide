---
title: ReentrantLock特性
tags:
  - Java/并发
  - 问答
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock特性

## Q1：ReentrantLock 有哪些核心特性？

**A**：ReentrantLock 基于 AQS 实现，核心特性包括：

1. **可重入**：同一线程可多次 `lock()`，内部用 `state` 计数，每次获取 +1，释放 -1，降至 0 才完全释放
2. **可中断**：`lockInterruptibly()` 在等待锁的过程中响应 `interrupt()`
3. **可超时**：`tryLock(timeout, unit)` 指定时间内未获取到锁则返回 false
4. **公平/非公平**：构造参数 `new ReentrantLock(true)` 为公平锁，默认非公平
5. **多条件变量**：`newCondition()` 可创建多个 Condition 对象，每个有独立的等待队列
6. **tryLock 非阻塞**：`tryLock()` 立即返回，不阻塞当前线程

---

## Q2：ReentrantLock 的内部结构是怎样的？

**A**：

```
ReentrantLock
├── sync: Sync（继承 AQS）
│   ├── NonfairSync（默认）
│   └── FairSync
└── 条件变量
```

- `ReentrantLock` 持有一个 `Sync` 类型的字段
- `Sync` 有两个子类：`NonfairSync` 和 `FairSync`
- 公平与非公平的差异体现在 `tryAcquire` 方法中：公平锁会检查 CLH 队列是否有等待节点，非公平锁直接 CAS 抢锁
- 每个 `Condition` 对象内部维护一个独立的等待队列

---

## Q3：ReentrantLock 的可重入是怎么实现的？

**A**：通过 AQS 的 `state` 字段记录重入次数，详细原理见 **AQS 12_可重入锁**。

简述：
- **首次获取**：`state` 从 0 CAS 变为 1，记录 exclusiveOwnerThread
- **同线程重入**：判断 `current == exclusiveOwnerThread`，`state +1`（无需 CAS）
- **释放**：`state -1`，降至 0 时才唤醒后继节点
- **不同线程**：CAS 失败，进入 CLH 队列阻塞等待

> 📖 **关联阅读**：[AQS 12_可重入锁](/11_并发/07_AQS/12_可重入锁/可重入锁.md) — 完整源码与 CLH 队列分析。

**A**：

| 方法 | 说明 |
|------|------|
| `getHoldCount()` | 当前线程持有锁的次数（重入计数） |
| `getQueueLength()` | 等待获取锁的线程数 |
| `getWaitQueueLength(Condition)` | 在指定条件上等待的线程数 |
| `hasQueuedThreads()` | 是否有线程在等待获取锁 |
| `hasWaiters(Condition)` | 是否有线程在指定条件上等待 |
| `isFair()` | 是否为公平锁 |
| `isHeldByCurrentThread()` | 锁是否被当前线程持有 |
| `isLocked()` | 锁是否被任何线程持有 |


