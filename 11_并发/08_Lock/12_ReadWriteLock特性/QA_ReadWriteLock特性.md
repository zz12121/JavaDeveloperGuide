---
title: ReadWriteLock特性
tags:
  - Java/并发
  - 问答
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# ReadWriteLock特性

## Q1：ReadWriteLock 的核心特性有哪些？

**A**：

1. **读读共享**：多个线程可同时持有读锁
2. **读写互斥**：有读锁时不能写，有写锁时不能读
3. **写写互斥**：同一时刻只有一个写线程
4. **可重入**：同一线程的读锁和写锁都可重入
5. **锁降级**：写锁 → 读锁 → 释放写锁（安全）
6. **不支持锁升级**：读锁 → 写锁（死锁）

---

## Q2：ReadWriteLock 适用于什么场景？

**A**：**读多写少**的场景。典型应用：

1. **本地缓存**：读缓存频率远高于更新缓存
2. **配置管理**：频繁读取配置，偶尔更新
3. **数据字典**：大量查询，少量更新

不适合场景：
- **写多**：频繁写操作导致读线程频繁阻塞，不如直接用 ReentrantLock
- **读多写也多**：写锁竞争激烈，ReadWriteLock 优势不明显

---

## Q3：ReadWriteLock 的读锁为什么能共享？

**A**：因为读操作不会修改数据，多个线程同时读不会产生数据不一致。AQS 层面，读锁使用**共享模式**（`acquireShared`），允许多个线程同时持有。

state 的高 16 位记录所有线程的读锁总持有次数，每次获取读锁 state += 65536。每个线程的读锁次数用 ThreadLocal（HoldCounter）单独记录。

---

## Q4：什么是锁降级？为什么要锁降级？

**A**：

**锁降级**流程：持有写锁 → 获取读锁 → 释放写锁

**目的**：保证在释放写锁后，仍然能读取到自己刚写入的数据，不会被其他线程的写操作覆盖。

如果不降级：
1. 释放写锁
2. 在获取读锁之前，其他线程可能已经修改了数据
3. 读取到的不是自己写入的最新值

通过锁降级，读锁在整个过程中一直持有，保证了数据一致性。

---

## Q5：ReadWriteLock 和 ReentrantLock 怎么选？

**A**：

| 条件 | 推荐 |
|------|------|
| 读多写少（读>90%） | ReadWriteLock |
| 读多写少（读>70%） | ReadWriteLock 或 StampedLock |
| 读写均衡 | ReentrantLock |
| 写多 | ReentrantLock |
| 需要精确控制读写 | ReadWriteLock |
| 简单场景 | synchronized |

---
```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// 读锁共享：多个线程可同时读
rwLock.readLock().lock();
try {
    String data = loadData();
} finally {
    rwLock.readLock().unlock();
}

// 写锁独占
rwLock.writeLock().lock();
try {
    updateData();
} finally {
    rwLock.writeLock().unlock();
}

// 锁降级：写锁 → 读锁 → 释放写锁
rwLock.writeLock().lock();
try {
    data = updateAndGet();  // 写入数据
    rwLock.readLock().lock();  // 获取读锁（降级前）
    try {
        return data;  // 释放写锁后仍能读到最新值
    } finally {
        rwLock.readLock().unlock();
    }
} finally {
    rwLock.writeLock().unlock();  // 释放写锁
}
```


