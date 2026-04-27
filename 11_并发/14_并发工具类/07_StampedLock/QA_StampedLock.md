
# StampedLock

## Q1：StampedLock 的乐观读是怎么实现的？

**A**：乐观读的核心是 **stamp 机制**：

1. `tryOptimisticRead()`：获取当前 state 的快照作为 stamp（类似版本号），**不加任何锁**
2. 读取数据：直接读取共享变量，完全无锁
3. `validate(stamp)`：校验当前 state 是否与 stamp 一致
   - 一致 → 期间没有写操作，读取结果有效
   - 不一致 → 期间有写操作，升级为悲观读锁重新读取

乐观读避免了读锁对写锁的阻塞，在读远多于写的场景下性能提升显著。

---

## Q2：StampedLock 和 ReentrantReadWriteLock 的区别？

**A**：

| 对比 | StampedLock | ReentrantReadWriteLock |
|------|------------|----------------------|
| 乐观读 | ✅ 核心特性 | ❌ 不支持 |
| 可重入 | ❌ | ✅ |
| Condition | ❌ | ✅ |
| 性能（读多写少） | 更高 | 较低 |
| API 风格 | 基于 stamp 参数 | lock/unlock 无参 |

**核心优势**：乐观读在读远多于写的场景下避免了读锁对写锁的阻塞。

---

## Q3：StampedLock 为什么不可重入？有什么影响？

**A**：因为 state 用 64 位同时存储锁状态和版本号，不支持线程所有权跟踪。

影响：同一个线程不能重复获取同一把锁。使用时需要在每个方法内部自行管理锁的获取和释放，不能嵌套调用。

---

## Q4：StampedLock 乐观读的使用模板？

**A**：这是面试高频考点：

```java
long stamp = sl.tryOptimisticRead();       // 获取 stamp
double x = this.x, y = this.y;             // 无锁读取
if (!sl.validate(stamp)) {                  // 校验
    stamp = sl.readLock();                  // 升级为悲观读锁
    try {
        x = this.x; y = this.y;             // 重新读取
    } finally {
        sl.unlockRead(stamp);
    }
}
// 使用 x, y
```


