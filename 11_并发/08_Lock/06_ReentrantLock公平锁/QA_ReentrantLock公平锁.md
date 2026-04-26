---
title: ReentrantLock公平锁
tags:
  - Java/并发
  - 问答
  - 源码型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock公平锁

## Q1：ReentrantLock 公平锁的实现原理是什么？

**A**：公平锁的核心在于 `tryAcquire` 中调用 `hasQueuedPredecessors()`：

```java
protected final boolean tryAcquire(int acquires) {
    if (c == 0) {
        // 公平锁关键：检查队列中是否有前驱节点
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // ... 重入逻辑
}
```

- `hasQueuedPredecessors()` 判断 CLH 队列中是否有比当前线程更早的等待者
- 有前驱 → 不抢锁，返回 false → 入队排队
- 无前驱 → CAS 尝试获取

---

## Q2：公平锁有什么性能问题？

**A**：

1. **吞吐量低于非公平锁**：每次获取锁都要检查队列（hasQueuedPredecessors），即使当前线程是下一个也必须经历完整的检查流程
2. **hasQueuedPredecessors 非原子操作**：读 head/tail 和读 next 之间可能有变化，需要 AQS 的 CAS 机制来保证整体正确性
3. **不能利用"空隙"**：非公平锁在锁释放瞬间可以直接 CAS 插队，而公平锁必须排队

实际场景中，**非公平锁性能通常优于公平锁 5%~10%**，除非有严格的公平性需求（如避免线程饥饿），否则不推荐使用公平锁。

---

## Q3：公平锁能完全保证公平吗？

**A**：不能绝对保证。原因：

1. `hasQueuedPredecessors()` 的判断和 `compareAndSetState()` 之间不是原子操作，可能判断时无前驱，CAS 时被插队
2. 线程从 park 唤醒到真正执行存在时间差，此期间可能被新线程 CAS 抢先
3. 但从统计意义上，公平锁能大幅减少饥饿现象


