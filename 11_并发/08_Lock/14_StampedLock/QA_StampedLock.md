---
title: StampedLock
tags:
  - Java/并发
  - 问答
  - 原理型
module: 08_Lock
created: 2026-04-26
---

# StampedLock（JDK8 读写锁改进版）

## Q1：StampedLock 和 ReadWriteLock 有什么区别？

**A**：核心区别在于读锁的实现方式：

| 维度 | ReadWriteLock（ReentrantReadWriteLock） | StampedLock |
|------|----------------------------------------|-------------|
| 读锁 | **悲观共享读**（持有读锁时阻塞写锁） | **乐观无锁读**（不阻塞写锁） |
| 写锁饥饿 | ❌ 有（读锁多时写锁等很久） | ✅ 无（乐观读不阻塞写入） |
| 可重入 | ✅ 可重入 | ❌ 不可重入 |
| 锁升级 | 不支持 | ✅ `tryConvertToWriteLock` |
| API | `readLock()`/`writeLock()` | `readLock()`/`writeLock()`/`tryOptimisticRead()` |
| 复杂度 | 简单 | 较高（需处理验证） |

```java
// ReadWriteLock：读锁阻塞写锁 → 写锁饥饿
readLock.lock();    // T1 持有读锁
readLock.lock();    // T2 也持有读锁
// writeLock 永远等待（饥饿）

// StampedLock：乐观读不阻塞写锁
long stamp = lock.tryOptimisticRead();  // 立即返回，不阻塞任何人
// 同时 writeLock 可以正常获取
```

---

## Q2：乐观读（tryOptimisticRead）是怎么工作的？

**A**：`tryOptimisticRead()` 不阻塞任何线程，返回一个"戳"（stamp），代表当前的数据版本。读取后必须通过 `validate(stamp)` 验证数据是否被修改：

```java
long stamp = lock.tryOptimisticRead();  // ① 获取版本戳
double dx = x;                           // ② 读取（可能不一致）
double dy = y;
if (!lock.validate(stamp)) {             // ③ 验证戳是否变化
    // 戳被修改（有写锁介入）→ 降级为悲观读
    stamp = lock.readLock();
    try {
        dx = x; dy = y;                  // 重新读取
    } finally {
        lock.unlockRead(stamp);
    }
}
return Math.sqrt(dx * dx + dy * dy);
```

**适用条件**：数据一致性要求不高（读到的稍微旧一点的数据可以接受）。

---

## Q3：为什么说 StampedLock 解决了写锁饥饿？

**A**：ReadWriteLock 中，读锁是共享的，写锁是独占的。当大量读线程持有读锁时，写锁永远无法获取（饥饿）。StampedLock 通过乐观读解决了这个问题：

```
ReadWriteLock（饥饿）：
  T1读 ─→ [持有读锁] ───────────────────────────────────┐
  T2读 ─→ [持有读锁] ───────────────────────────────────┤
  T3读 ─→ [持有读锁] ───────────────────────────────────┼→ 写锁等不到
  T写 ─→ [等待写锁] ───────────────────────────────────┘

StampedLock（无饥饿）：
  T1乐观读 ──→ [获取戳] ──→ [读取数据] ──→ [验证通过] ✓
  T2写锁   ──→ [同时获取写锁] ──→ [写入数据] ✓
  T3乐观读 ──→ [获取戳] ──→ [读取数据] ──→ [验证失败/通过]
```

乐观读不持有任何锁，不阻塞写锁，所以写锁不会被饥饿。

---

## Q4：StampedLock 的锁升级怎么实现？

**A**：通过 `tryConvertToWriteLock()` 尝试将读锁升级为写锁：

```java
long stamp = lock.readLock();
try {
    // 读取数据
    if (条件满足) {
        long ws = lock.tryConvertToWriteLock(stamp);  // 尝试升级
        if (ws != 0) {
            stamp = ws;    // 升级成功，继续写
            // 写入数据
        } else {
            // 升级失败（有其他读锁）→ 手动重试
            lock.unlockRead(stamp);
            stamp = lock.writeLock();
            try {
                // 写入数据
            } finally {
                lock.unlockWrite(stamp);
            }
            return;
        }
    }
} finally {
    lock.unlockRead(stamp);
}
```

**注意**：`tryConvertToWriteLock` 失败的原因是有其他线程持有读锁，此时需要释放读锁再获取写锁。

---

## Q5：StampedLock 为什么不支持重入？

**A**：StampedLock 的设计目标是大规模并发读写。为支持可重入，需要在 state 中额外记录持有线程 ID（类似 AQS 的 exclusiveOwnerThread）。这会增加复杂性并损失性能，因此 StampedLock **故意设计为不可重入**。

```java
// 错误：StampedLock 不可重入
long stamp = lock.writeLock();
try {
    long stamp2 = lock.writeLock();  // ⚠️ 死锁！
    // ...
} finally {
    lock.unlockWrite(stamp);
}
```

如果需要可重入特性，使用 `ReentrantReadWriteLock`。

---

## Q6：StampedLock 的 validate(stamp) 验证原理是什么？

**A**：`validate(stamp)` 检查从 `tryOptimisticRead()` 到调用 `validate()` 期间，是否有写锁操作（修改了 stamp）：

```java
public boolean validate(long stamp) {
    // 将 stamp 与当前的 state 比较
    // stamp 中包含了写锁的版本信息
    // 如果期间发生了写锁获取/释放，版本号变化，验证失败
    return (stamp & SBITS) == (state & SBITS);
}
```

**验证失败的处理策略**：
1. **降级为悲观读**（推荐）：`tryConvertToReadLock` 或 `readLock()`
2. **重试乐观读**：再次调用 `tryOptimisticRead()`，适合数据一致性要求不高的场景
3. **直接报错**：适合一致性要求极高的场景

---

## Q7：StampedLock 适合什么场景？

**A**：最适合**读多写少且不允许写饥饿**的场景：

| 场景 | 推荐锁 |
|------|--------|
| 读 >> 写，不允许写饥饿 | ✅ StampedLock（乐观读） |
| 读 >> 写，可以接受写饥饿 | ReadWriteLock（简单） |
| 写为主，偶尔读取 | StampedLock（写锁）或 synchronized |
| 读一致性要求极高 | ReadWriteLock（悲观读） |
| 需要可重入 | ReadWriteLock（StampedLock 不支持） |
| 缓存（读多写少） | ✅ StampedLock（乐观读 + validate） |
