---
title: ReentrantLock非公平锁
tags:
  - Java/并发
  - 源码型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock非公平锁

## 核心结论

非公平锁是 ReentrantLock 的默认模式。核心特点：**获取锁时先 CAS 插队抢一次，失败才入队排队**。减少了线程切换开销，吞吐量高于公平锁。

## 深度解析

### NonfairSync 源码

```java
// NonfairSync.lock()
final void lock() {
    // 第一步：直接 CAS 抢锁（插队）
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 第二步：抢失败，走正常 AQS 流程
        acquire(1); // → tryAcquire → addWaiter → acquireQueued
}

// NonfairSync.tryAcquire
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

// Sync.nonfairTryAcquire（非公平锁和公平锁共享）
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 注意：这里没有 hasQueuedPredecessors() 检查
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 非公平 vs 公平关键差异

```
非公平锁 lock():
  1. compareAndSetState(0,1)  ← 直接插队抢
     ↓ 失败
  2. acquire → tryAcquire
     ↓ state==0 时直接 CAS（不检查队列）
     ↓ 失败
  3. addWaiter → acquireQueued（入队自旋）

公平锁 lock():
  1. acquire → tryAcquire
     ↓ state==0 时先 hasQueuedPredecessors()（检查队列）
     ↓ 有前驱则不抢
  2. addWaiter → acquireQueued（入队自旋）
```

### 非公平锁的"两次插队"机会

1. **第一次**：`lock()` 方法内直接 CAS 抢 state
2. **第二次**：`tryAcquire()` 中 state== 0 时再次 CAS（不检查队列）

即使队列中有等待线程，新来的线程也有两次机会插队成功。

### 为什么非公平锁性能更好

1. **减少线程切换**：唤醒队列中的线程需要 unpark（内核态操作），而新线程在用户态直接 CAS
2. **利用 CPU 缓存**：刚释放锁的线程的 CPU 缓存还热，但新线程可能也已经在运行状态
3. **降低唤醒开销**：省去了 park/unpark 的上下文切换代价

### 饥饿风险

- 非公平锁在高并发场景下，队列中的线程可能长期等不到锁（线程饥饿）
- 但实际生产中饥饿概率较低，且非公平锁的吞吐量优势通常更重要

## 关联知识点
