---
title: StampedLock vs ReadWriteLock
tags:
  - Java/并发
  - 问答
  - 对比型
module: 14_并发工具类
created: 2026-04-18
---

# StampedLock vs ReadWriteLock

## Q1：StampedLock 相比 ReentrantReadWriteLock 的核心优势？

**A**：**乐观读**。

ReentrantReadWriteLock 的读操作需要获取读锁，虽然读读共享，但读锁会阻塞写锁。在大量读线程存在时，写线程可能长时间获取不到锁。

StampedLock 的乐观读不加锁，只获取一个版本号（stamp），读完校验。如果期间没有写操作，读取完全无锁，写操作也不被阻塞。

---

## Q2：什么时候不应该用 StampedLock？

**A**：

1. **需要可重入**：同一线程可能嵌套获取锁 → 用 ReentrantReadWriteLock
2. **需要 Condition**：需要 await/signal → 用 ReentrantReadWriteLock
3. **团队不熟悉**：StampedLock 使用复杂，容易出错
4. **写操作频繁**：乐观读优势不明显

---

## Q3：StampedLock 乐观读失败怎么办？

**A**：升级为悲观读锁重新读取：

```java
long stamp = sl.tryOptimisticRead();
double x = this.x; // 无锁读
if (!sl.validate(stamp)) {
    stamp = sl.readLock(); // 升级
    try {
        x = this.x;
    } finally {
        sl.unlockRead(stamp);
    }
}
```

乐观读失败只是多了一次重试的代价，不会丢失正确性。

