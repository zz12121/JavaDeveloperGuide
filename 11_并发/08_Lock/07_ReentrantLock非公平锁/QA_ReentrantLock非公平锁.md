---
title: ReentrantLock非公平锁
tags:
  - Java/并发
  - 问答
  - 源码型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock非公平锁

## Q1：非公平锁的 lock() 方法和公平锁有什么不同？

**A**：非公平锁的 `lock()` 方法会先**直接 CAS 插队**：

```java
// NonfairSync.lock()
final void lock() {
    if (compareAndSetState(0, 1))  // 插队抢
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);  // 抢失败才走 AQS 流程
}
```

而公平锁的 `lock()` 直接调用 `acquire(1)`，进入 `tryAcquire` 时会先检查队列。

所以非公平锁有**两次插队机会**：
1. `lock()` 内的 `compareAndSetState`
2. `tryAcquire` 中 `state==0` 时直接 CAS（不检查队列）

---

## Q2：为什么默认用非公平锁？

**A**：

1. **性能更高**：不需要调用 `hasQueuedPredecessors()` 检查队列
2. **减少上下文切换**：刚释放锁时，新线程直接 CAS 比 unpark 等待线程更快
3. **吞吐量更大**：实测非公平锁吞吐量比公平锁高约 5%~10%
4. **饥饿概率低**：实际场景中，即使有插队，队列中的线程最终也能获取锁

---

## Q3：非公平锁可能导致什么问题？

**A**：主要问题是**线程饥饿**。在高并发场景下：

- 新来的线程不断插队成功
- 队列中等待的线程迟迟获取不到锁
- 极端情况下，某个线程可能长时间无法执行

解决方案：
- 使用公平锁 `new ReentrantLock(true)`
- 使用带超时的 `tryLock(timeout)` 避免无限等待
- 合理设置线程优先级

但在大多数业务场景中，非公平锁的饥饿风险可以接受，而性能优势更有价值。

