---
title: Lock接口
tags:
  - Java/并发
  - 问答
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# Lock接口（lock/unlock/tryLock/tryLock(timeout)/lockInterruptibly/newCondition）

## Q1：Lock 接口有哪些方法？各自的特点是什么？

**A**：

| 方法 | 特点 |
|------|------|
| `lock()` | 阻塞获取锁，不响应中断 |
| `lockInterruptibly()` | 阻塞获取锁，等待时可被 `interrupt()` 中断 |
| `tryLock()` | 非阻塞，立即返回是否获取成功 |
| `tryLock(timeout)` | 超时阻塞，超时或中断后返回 false |
| `unlock()` | 释放锁 |
| `newCondition()` | 创建条件变量 |

---

## Q2：lock() 和 lockInterruptibly() 有什么区别？

**A**：

- `lock()`：线程等待锁时**不响应中断**，会一直等到获取锁
- `lockInterruptibly()`：线程等待锁时**响应中断**，`interrupt()` 会抛出 `InterruptedException`

```java
// lock() — 中断无法打断等待
lock.lock(); // 即使被 interrupt 也会继续等

// lockInterruptibly() — 中断可以打断等待
try {
    lock.lockInterruptibly(); // 被中断时抛出 InterruptedException
} catch (InterruptedException e) {
    // 处理中断
}
```

---

## Q3：tryLock() 有什么实际用途？

**A**：主要用于**避免死锁**和**非阻塞式资源竞争**：

```java
// 避免死锁：获取多个锁时，如果任一获取失败则释放已获取的
if (lockA.tryLock()) {
    try {
        if (lockB.tryLock()) {
            try { /* 成功 */ }
            finally { lockB.unlock(); }
        }
    } finally { lockA.unlock(); }
}
```

`tryLock()` 返回 false 时不阻塞，可以灵活处理失败情况（重试、降级、跳过）。

---

## Q4：LockSupport 的 park/unpark 机制是怎样的？

**A**：`LockSupport` 是 JUC 锁的底层支撑，每个线程有一个 permit（0 或 1）：

| 操作 | permit=0（无令牌） | permit=1（有令牌） |
|------|-------------------|-------------------|
| park() | **阻塞等待** | 消费令牌，立即返回 |
| unpark(t) | permit=1，唤醒 t | permit 仍为 1（不累积） |

```java
// 特点：unpark 可以在 park 之前调用
LockSupport.unpark(t);     // 先给令牌
LockSupport.park();        // 已有令牌，立即返回（不阻塞）

// 相比 wait/notify 的优势：
// wait/notify 必须先持有 monitor 才能调用
// park/unpark 无此限制
```

---

## Q5：park 和 Thread.sleep 有什么区别？

**A**：主要区别在于释放 CPU 和恢复方式：

| 维度 | LockSupport.park | Thread.sleep |
|------|-------------------|--------------|
| 响应中断 | ✅ 抛出 InterruptedException | ✅ 中断后立即返回 |
| 恢复方式 | unpark() 或中断 | 自动唤醒（sleep 时间到） |
| 释放 CPU | ✅ park 时让出 | ✅ sleep 时让出 |
| 底层 | OS 层面的 park（Unsafe） | JVM 实现 |
| 是否释放锁 | 不涉及（需配合 Lock） | 不释放任何锁 |

**重要**：`LockSupport.park()` 必须配合 Lock 使用（由 Lock 保证原子性和可见性），而 `Thread.sleep()` 不释放锁，在持有锁时 sleep 会导致其他线程永远等待。


