
# QA_ReadWriteLock vs StampedLock

## Q1: ReadWriteLock 和 StampedLock 的核心区别是什么？

**答：**

| 区别 | ReentrantReadWriteLock | StampedLock |
|------|----------------------|-------------|
| 读锁类型 | 悲观读（持有锁） | 悲观读 + 乐观读 |
| 可重入 | ✅ 完全支持 | ❌ 不支持 |
| Condition | ✅ 支持 | ❌ 不支持 |
| 锁降级 | ✅ 支持（写→读） | ❌ 不支持 |
| 乐观读 | ❌ 无 | ✅ 有 |

**核心区别**：
- **ReadWriteLock**：悲观读写，读锁共享，写锁独占
- **StampedLock**：增加乐观读模式，读时先不加锁，读完验证stamp是否有效

---

## Q2: 什么场景下应该用 StampedLock？

**答：**

**适用场景**：
1. **读多写少**（读写比例 > 10:1）
2. **性能敏感**（高并发读操作）
3. **不需要 Condition**（StampedLock不支持）
4. **不需要重入**（StampedLock不可重入）

**不适用场景**：
1. 读写比例接近 1:1
2. 需要可重入
3. 需要 Condition 等待

```java
// 高性能读场景示例
StampedLock lock = new StampedLock();
long stamp = lock.tryOptimisticRead();  // 乐观读，不阻塞
// 读数据
if (!lock.validate(stamp)) {  // 验证
    // 失败，升级为悲观读
    stamp = lock.readLock();
    try {
        // 读数据
    } finally {
        lock.unlockRead(stamp);
    }
}
```

---

## Q3: StampedLock 的乐观读是什么原理？

**答：**

```
1. tryOptimisticRead() 获取 stamp（一个长整型）
   - 不阻塞，直接返回 stamp
   - 此时没有任何线程持有锁

2. 读取数据（可能读到脏数据）

3. validate(stamp) 验证 stamp 是否有效
   - 如果有效，说明读期间没有写操作，数据是干净的
   - 如果无效，说明有写操作发生，需要重读

4. 如果验证失败，升级为悲观读重试
```

**乐观读优势**：
- 不涉及锁操作，性能好
- 无线程切换、无CAS竞争

**乐观读劣势**：
- 可能需要重试（读失败）
- 不适合写频繁的场景

---

## Q4: 为什么 StampedLock 不支持 Condition？

**答：**

**技术原因**：
1. StampedLock 基于 CLH 队列变体，不是标准的 AQS
2. Condition 需要 Lock 和 AQS 的深度整合
3. StampedLock 的设计目标是极致性能，不支持 Condition 是设计取舍

**替代方案**：
```java
// 需要 Condition 的场景，用 ReentrantReadWriteLock
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
Condition condition = rwLock.writeLock().newCondition();

// 等待条件
condition.await();

// 唤醒
condition.signal();
```

---

## Q5: StampedLock 不可重入会导致什么问题？

**答：**

**死锁问题**：
```java
StampedLock lock = new StampedLock();

public void doSomething() {
    long stamp = lock.writeLock();
    try {
        doWrite();
        // ❌ 递归调用会死锁
        doSomethingElse();  // 再次获取写锁 → 死锁
    } finally {
        lock.unlockWrite(stamp);
    }
}

public void doSomethingElse() {
    long stamp = lock.writeLock();  // 死锁点
    try {
        // ...
    } finally {
        lock.unlockWrite(stamp);
    }
}
```

**解决方案**：
1. 使用 ReentrantReadWriteLock（可重入）
2. 重构代码，避免同一个锁的递归调用

---

## Q6: StampedLock 如何实现锁降级？

**答：**

**StampedLock 不支持直接的锁降级**（写→读）。

但可以通过**乐观读 + 悲观读**模拟：
```java
StampedLock lock = new StampedLock();

long stamp = lock.writeLock();  // 获取写锁
try {
    write();
    // 释放写锁前尝试乐观读验证
    long optimisticStamp = lock.tryOptimisticRead();
    if (lock.validate(optimisticStamp)) {
        // 写操作后数据一致，可以继续
    }
} finally {
    lock.unlockWrite(stamp);
}

// 如果需要降级为读锁持有的状态，用 ReentrantReadWriteLock
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.writeLock().lock();
try {
    write();
    rwLock.readLock().lock();  // 获取读锁
} finally {
    rwLock.writeLock().unlock();  // 释放写锁 → 锁降级
}
// 此时持有读锁
```

---

## Q7: ReadWriteLock 和 StampedLock 的性能差异？

**答：**

**性能对比**（读:写 = 100:1，100万次操作）：

| 锁类型 | 耗时 | 吞吐量 |
|--------|------|--------|
| 无锁 | 50ms | 最高 |
| StampedLock（乐观读） | 80ms | 高 |
| ReentrantReadWriteLock | 150ms | 中 |
| synchronized | 200ms | 较低 |

**影响因素**：
1. 读写比例：读越多，StampedLock 优势越大
2. 冲突频率：冲突多时乐观读会重试，优势减小
3. 业务逻辑复杂度：读操作越简单，乐观读收益越高

---

## Q8: 如何选择 ReadWriteLock 和 StampedLock？

**答：**

```
开始
  ↓
需要 Condition？
  → 是 → ReadWriteLock ✅
  ↓ 否
需要可重入？
  → 是 → ReadWriteLock ✅
  ↓ 否
读:写 比例 > 10:1 且性能敏感？
  → 是 → StampedLock ✅
  ↓ 否
追求极致性能优化？
  → 是 → StampedLock ✅
  ↓ 否
写操作多或需要复杂锁控制？
  → 是 → ReadWriteLock ✅
  ↓ 默认
一般场景 → ReadWriteLock（更简单安全）
```

**实战建议**：
```java
// 简单业务场景
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

// 高性能读多写少场景
private final StampedLock stampedLock = new StampedLock();
```
