---
title: lockInterruptibly
tags:
  - Java/并发
  - 问答
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# lockInterruptibly

## Q1：lockInterruptibly() 和 lock() 的区别是什么？

**A**：核心区别在于**中断处理方式**：

- `lock()`：等待期间被 `interrupt()`，只恢复中断标志，**继续等待获取锁**
- `lockInterruptibly()`：等待期间被 `interrupt()`，立即抛出 `InterruptedException`，**退出等待**

源码差异：
```java
// lock() → acquireQueued
parkAndCheckInterrupt() → selfInterrupt(); // 恢复标志，继续等

// lockInterruptibly() → doAcquireInterruptibly
parkAndCheckInterrupt() → throw new InterruptedException(); // 直接退出
```

---

## Q2：lockInterruptibly() 在什么场景下使用？

**A**：

1. **可取消的任务**：线程池中提交的任务，调用 `future.cancel(true)` 可中断等待
2. **优雅关闭**：应用关闭时，需要中断正在等待锁的线程
3. **避免死锁**：多锁场景下，一个锁等待太久时可以被中断，从而释放其他已持有的锁
4. **超时替代方案**：配合 `Thread.interrupt()` 实现外部可控的"超时"

---

## Q3：lockInterruptibly() 被中断后需要做什么？

**A**：

1. 捕获 `InterruptedException`
2. **恢复中断标志**：`Thread.currentThread().interrupt()`（因为中断标志在抛异常时被清除）
3. 确保已持有的锁被释放（如果有的话）
4. 执行清理或回滚逻辑

```java
try {
    lock.lockInterruptibly();
    try {
        doWork();
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断标志
    cleanup();
}
```


