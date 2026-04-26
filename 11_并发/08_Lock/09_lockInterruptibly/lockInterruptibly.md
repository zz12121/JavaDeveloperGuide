---
title: lockInterruptibly
tags:
  - Java/并发
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# lockInterruptibly

## 核心结论

`lockInterruptibly()` 获取锁的过程中**响应线程中断**。当等待获取锁的线程被其他线程调用 `interrupt()` 时，抛出 `InterruptedException`，避免死锁。

## 深度解析

### 源码

```java
// ReentrantLock
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

// AQS
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);  // 与 acquireQueued 不同
}

// AQS.doAcquireInterruptibly
private void doAcquireInterruptibly(int arg) {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())  // 被中断返回 true
                throw new InterruptedException();  // 直接抛异常
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 与 lock() 的关键区别

```java
// lock() 的 acquireQueued：
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    selfInterrupt();  // 只是恢复中断标志，继续等待

// lockInterruptibly() 的 doAcquireInterruptibly：
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    throw new InterruptedException();  // 直接抛异常退出
```

- `lock()` 被中断后**继续等待获取锁**，只是恢复中断标志
- `lockInterruptibly()` 被中断后**立即退出等待**，抛出异常

### 使用场景

```java
ReentrantLock lock = new ReentrantLock();

// 场景1：可取消的任务
public void cancellableTask() {
    try {
        lock.lockInterruptibly();
        try {
            // 长时间操作，可能需要被取消
            longOperation();
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        log.info("任务被取消");
    }
}

// 场景2：线程池任务中断
Future<?> future = executor.submit(() -> {
    try {
        lock.lockInterruptibly();
        try {
            doWork();
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

// 取消任务
future.cancel(true); // 中断线程 → lockInterruptibly 抛异常
```

### 三种获取方式对比

| 方法 | 中断处理 | 超时 | 阻塞行为 |
|------|---------|------|---------|
| `lock()` | 忽略中断（恢复标志） | 无 | 阻塞直到获取 |
| `lockInterruptibly()` | 抛 InterruptedException | 无 | 阻塞直到获取或中断 |
| `tryLock(timeout)` | 抛 InterruptedException | 支持 | 阻塞直到获取/超时/中断 |

## 关联知识点

