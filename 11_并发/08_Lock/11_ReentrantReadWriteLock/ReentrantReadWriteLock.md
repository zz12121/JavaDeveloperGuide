---
title: ReentrantReadWriteLock
tags:
  - Java/并发
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# ReentrantReadWriteLock

## 核心结论

ReentrantReadWriteLock 是读写锁实现：**读锁共享、写锁互斥**。内部用一个 `state` 的高 16 位表示读锁持有次数（共享计数），低 16 位表示写锁重入次数（独占计数）。适合**读多写少**场景。

## 深度解析

### 内部结构

```
ReentrantReadWriteLock
├── sync: Sync（AQS 子类）
│   ├── FairSync
│   └── NonfairSync（默认）
├── readerLock: ReadLock（共享锁）
└── writerLock: WriteLock（独占锁）
```

### state 设计

```
state (int 32位)
┌──────────────┬──────────────┐
│  高16位: 读锁 │  低16位: 写锁 │
│  (共享计数)   │  (独占计数)   │
└──────────────┴──────────────┘
```

```java
static final int SHARED_SHIFT = 16;
static final int SHARED_UNIT = 1 << SHARED_SHIFT;  // 65536
static final int MAX_COUNT = (1 << SHARED_SHIFT) - 1; // 65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

// 读锁计数
static int sharedCount(int c) { return c >>> SHARED_SHIFT; }
// 写锁计数
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

### 锁关系矩阵

| | 读锁 | 写锁 |
|---|------|------|
| **读锁** | ✅ 兼容（共享） | ❌ 互斥 |
| **写锁** | ❌ 互斥 | ❌ 互斥 |

- **读读共享**：多个线程可同时持有读锁
- **读写互斥**：读锁和写锁不能同时持有
- **写写互斥**：同一时刻只有一个写锁

### 读写锁源码

state 分片设计、读锁 tryAcquireShared 完整源码、写锁 tryAcquire 完整源码见 **AQS 15_ReentrantReadWriteLock**。

核心要点：
- **读锁**（共享）：用 `acquireShared`，`state` 高 16 位累加，`HoldCounter` 记录每个线程的读锁重入
- **写锁**（独占）：用 `acquire`，`state` 低 16 位记录，写锁排斥一切
- **公平 vs 非公平**：公平锁在 `writerShouldBlock` / `readerShouldBlock` 中调用 `hasQueuedPredecessor`

> 📖 **关联阅读**：[AQS 15_ReentrantReadWriteLock](/11_并发/07_AQS/15_ReentrantReadWriteLock/ReentrantReadWriteLock.md) — 完整 state 分片源码、读写互斥逻辑、锁降级示例。

```java
// 锁降级：持有写锁 → 获取读锁 → 释放写锁
lock.writeLock().lock();
try {
    // 写操作
    data = updateData();
    lock.readLock().lock();  // 在持有写锁时获取读锁
    try {
        // 读取刚写入的数据（保证可见性）
    } finally {
        lock.writeLock().unlock(); // 释放写锁，降级为读锁
    }
    // 继续持有读锁，安全读取
} finally {
    lock.readLock().unlock();
}
```

**支持锁降级，不支持锁升级**（持有读锁时不能获取写锁，否则死锁）。

## 关联知识点
