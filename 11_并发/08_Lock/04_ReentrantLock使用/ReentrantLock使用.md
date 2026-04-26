---
title: ReentrantLock使用
tags:
  - Java/并发
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock使用

## 核心结论

使用 ReentrantLock 必须**在 finally 块中释放锁**，防止异常导致锁永远不释放。推荐使用 `try-finally` 模式，而非 `try-lock-finally`。

## 深度解析

### 基本使用模式

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();          // 获取锁（不可中断，无超时）
try {
    // 临界区操作
} finally {
    lock.unlock();    // 必须在 finally 中释放
}
```

### 标准错误处理

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // 临界区
    if (!condition) {
        return;      // 提前返回时，finally 仍会执行 unlock
    }
    // 其他操作
} finally {
    lock.unlock();   // 保证锁一定会被释放
}
```

### 各 API 使用场景

```java
ReentrantLock lock = new ReentrantLock();

// 1. lock() — 不可中断
lock.lock();

// 2. tryLock() — 非阻塞尝试
if (lock.tryLock()) {
    try {
        // 获取成功，执行操作
    } finally {
        lock.unlock();
    }
} else {
    // 获取失败，执行其他逻辑（如降级处理）
}

// 3. tryLock(timeout) — 超时尝试
try {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        try {
            // 5秒内获取成功
        } finally {
            lock.unlock();
        }
    } else {
        // 超时未获取到
    }
} catch (InterruptedException e) {
    // 等待过程中被中断
    Thread.currentThread().interrupt();
}

// 4. lockInterruptibly() — 可中断等待
try {
    lock.lockInterruptibly();
    try {
        // 临界区
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    // 获取锁时被中断
    Thread.currentThread().interrupt();
}
```

### 注意事项

1. **unlock 必须在 finally 中**：任何异常、return、break 都不会跳过 finally
2. **lock 前不要 try**：`lock()` 本身不会抛异常，不需要 try 包裹
3. **不要对未锁定的锁调用 unlock**：会抛 `IllegalMonitorStateException`
4. **注意可重入次数匹配**：lock 几次就要 unlock 几次
5. **避免在锁内执行耗时操作**：会阻塞其他等待线程

## 关联知识点