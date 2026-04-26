---
title: tryLock
tags:
  - Java/并发
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# tryLock

## 核心结论

`tryLock()` 是 ReentrantLock 提供的非阻塞锁获取方法。调用后立即返回 true/false，**不阻塞当前线程**。适合避免死锁、降级处理等场景。

## 深度解析

### API 签名

```java
// 无参：立即尝试获取，获取不到立即返回 false
boolean tryLock()

// 有参：在指定时间内尝试获取，超时返回 false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException
```

### 无参 tryLock 源码

```java
// ReentrantLock.tryLock()
public boolean tryLock() {
    return sync.nonfairTryAcquire(1); // 注意：直接调用非公平版本的 tryAcquire
}
```

- 即使是公平锁创建的 ReentrantLock，`tryLock()` 也不检查队列（非公平行为）
- 因为 tryLock 的语义是"尝试获取"，如果还要排队就失去了非阻塞的意义

### 有参 tryLock 实现

```java
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(time));
}

// AQS.tryAcquireNanos
public final boolean tryAcquireNanos(int arg, long nanosTimeout) {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        parkNanos(this, nanosTimeout) &&  // 限时 park
        (Thread.interrupted() || tryAcquire(arg)); // 唤醒后再试
}
```

### 典型场景

#### 1. 避免死锁（链式获取锁）

```java
ReentrantLock lockA = new ReentrantLock();
ReentrantLock lockB = new ReentrantLock();

// 按固定顺序获取可避免死锁，但 tryLock 更灵活
while (true) {
    if (lockA.tryLock()) {
        try {
            if (lockB.tryLock()) {
                try {
                    // 同时持有 lockA 和 lockB
                    return;
                } finally {
                    lockB.unlock();
                }
            }
        } finally {
            lockA.unlock();
        }
    }
    Thread.sleep(100); // 退避
}
```

#### 2. 降级处理

```java
if (lock.tryLock()) {
    try {
        // 获取到锁，执行操作
    } finally {
        lock.unlock();
    }
} else {
    // 获取失败，执行降级逻辑
    fallbackMethod();
}
```

#### 3. 定时任务超时保护

```java
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        // 必须在3秒内完成的操作
    } finally {
        lock.unlock();
    }
} else {
    log.warn("获取锁超时，跳过本次任务");
}
```

# tryLock(timeout)

## 核心结论

`tryLock(timeout)` 在指定时间内等待获取锁，超时返回 false。等待过程中**可响应中断**（抛 InterruptedException），使用 `LockSupport.parkNanos()` 实现限时阻塞。

## 深度解析

### 源码链路

```java
// ReentrantLock
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(time));
}

// AQS
public final boolean tryAcquireNanos(int arg, long nanosTimeout) {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        parkNanos(this, nanosTimeout) &&
        (Thread.interrupted() || tryAcquire(arg));
}

// AQS.parkNanos
private static boolean parkNanos(Object blocker, long nanos) {
    if (nanos > 0)
        LockSupport.parkNanos(blocker, nanos);
    return Thread.interrupted();
}
```

### 执行流程

```
tryLock(5, SECONDS)
  → Thread.interrupted() ? → 抛 InterruptedException
  → tryAcquire(1)
    → 成功 → 返回 true
    → 失败 → 继续
  → parkNanos(5秒)
    → 期间被释放锁的线程 unpark 唤醒
      → Thread.interrupted() ? → 抛 InterruptedException
      → tryAcquire(1)
        → 成功 → 返回 true
        → 失败 → 继续自旋
    → 5秒到期自动唤醒
      → tryAcquire(1) → 成功返回 true / 失败返回 false
```

### 超时机制

- 使用 `LockSupport.parkNanos()` 实现精确的纳秒级超时
- 超时后自动唤醒，再次尝试 `tryAcquire`
- 如果此时 state 刚好为 0，可能获取成功

### 中断处理

```java
try {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        try {
            // 获取成功
        } finally {
            lock.unlock();
        }
    } else {
        // 超时
    }
} catch (InterruptedException e) {
    // 等待过程中被中断
    Thread.currentThread().interrupt(); // 恢复中断标志
    log.warn("获取锁被中断");
}
```

### 与 lockInterruptibly 的区别

| 维度 | tryLock(timeout) | lockInterruptibly |
|------|-----------------|-------------------|
| 超时 | 支持超时返回 | 无超时，一直等待 |
| 中断 | 支持 | 支持 |
| 失败返回 | false | 抛 InterruptedException |
| 使用场景 | 需要超时控制 | 不需要超时，但要可中断 |

## 关联知识点

