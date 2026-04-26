---
title: tryLock
tags:
  - Java/并发
  - 问答
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# tryLock

## Q1：tryLock() 和 lock() 有什么区别？

**A**：

| 方法 | 阻塞行为 | 响应中断 | 检查队列 |
|------|---------|---------|---------|
| `lock()` | 阻塞直到获取 | 不响应 | 公平锁检查 |
| `tryLock()` | 不阻塞，立即返回 | 不涉及 | **不检查（即使公平锁）** |

`tryLock()` 直接调用 `nonfairTryAcquire()`，即使是公平锁也不检查 CLH 队列。

---

## Q2：tryLock() 为什么在公平锁上也不检查队列？

**A**：因为 `tryLock()` 的语义是"尝试获取一次，失败就放弃"。如果还要检查队列并等待，就变成了阻塞式获取，与 `tryLock` 的非阻塞语义矛盾。

如果需要在公平模式下"尝试按队列顺序获取"，应使用 `tryLock(0, TimeUnit.SECONDS)` 或直接用 `lock()`。

---

## Q3：tryLock 在什么场景下特别有用？

**A**：

1. **避免死锁**：需要同时获取多个锁时，tryLock 失败后释放已持有的锁并重试
2. **降级处理**：获取锁失败时执行备用逻辑（如返回缓存数据、使用默认值）
3. **超时保护**：定时任务中限制锁等待时间，避免任务积压
4. **探针模式**：检查锁是否可用，但不阻塞（`lock.isLocked()` 的替代）

---
```java
ReentrantLock lock = new ReentrantLock();

// tryLock：非阻塞，立即返回
if (lock.tryLock()) {
    try {
        // 获取锁成功
        doWork();
    } finally {
        lock.unlock();
    }
} else {
    // 获取锁失败，执行降级逻辑
    log.warn("获取锁失败，返回缓存数据");
}

// tryLock 避免死锁示例
Lock lockA = new ReentrantLock();
Lock lockB = new ReentrantLock();
while (true) {
    if (lockA.tryLock()) {
        try {
            if (lockB.tryLock()) {
                try { /* 同时持有两把锁 */ }
                finally { lockB.unlock(); }
                break;
            }
        } finally { lockA.unlock(); }
    }
    Thread.yield();  // 短暂让步后重试
}
```

---

# tryLock(timeout)

## Q1：tryLock(timeout) 的工作流程是怎样的？

**A**：

1. 检查中断标志，已中断则抛 `InterruptedException`
2. 调用 `tryAcquire()` 尝试获取，成功返回 true
3. 失败则调用 `LockSupport.parkNanos()` 限时阻塞
4. 被 unpark 唤醒或超时后，再次检查中断并尝试 `tryAcquire()`
5. 多次自旋直到获取成功或超时返回 false

核心用 `parkNanos` 实现精确超时，用 `Thread.interrupted()` 检测中断。

---

## Q2：tryLock(timeout) 和 lockInterruptibly() 有什么区别？

**A**：

- **tryLock(timeout)**：超时后返回 false，不抛异常；等待过程中被中断抛 `InterruptedException`
- **lockInterruptibly()**：无超时，一直等到获取锁或被中断抛 `InterruptedException`

简单说：tryLock(timeout) 多了超时返回 false 的能力，lockInterruptibly 只能通过中断退出等待。

---

## Q3：tryLock(timeout) 超时后锁状态是怎样的？

**A**：超时返回 false 时，当前线程**没有持有锁**。超时只是等待结束，不会影响锁的 state。如果超时期间其他线程释放了锁，但当前线程的 parkNanos 还没到期，当前线程被唤醒后可能获取成功。

所以超时返回 false 意味着：在 timeout 时间内没有获取到锁，不保证锁当前是否空闲。

---
```java
ReentrantLock lock = new ReentrantLock();

try {
    // 最多等 3 秒
    if (lock.tryLock(3, TimeUnit.SECONDS)) {
        try {
            doWork();
        } finally {
            lock.unlock();
        }
    } else {
        // 超时未获取锁
        log.warn("获取锁超时");
    }
} catch (InterruptedException e) {
    // 等待过程中被中断
    Thread.currentThread().interrupt();
    log.warn("获取锁被中断");
}
```

