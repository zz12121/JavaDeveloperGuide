---
title: synchronized特性
tags:
  - Java/并发
  - 问答
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# synchronized特性（可重入/不可中断/阻塞式/独享/互斥）

## Q1：synchronized 有哪些核心特性？

**A**：synchronized 具备五大核心特性：

1. **可重入**：同一线程可多次获取同一把锁，不会死锁。JVM 通过记录持有线程 ID 和重入次数实现。
2. **不可中断**：线程等待 synchronized 锁时无法响应 `Thread.interrupt()`。
3. **阻塞式**：竞争不到锁的线程进入 BLOCKED 状态，涉及内核态切换。
4. **独享**：同一时刻只有一个线程能持有该锁。
5. **互斥**：其他线程必须等待当前线程释放锁。

---

## Q2：synchronized 能保证哪些并发特性？

**A**：synchronized 同时保证**原子性、可见性、有序性**：

- **原子性**：同步代码块同一时刻只允许一个线程执行
- **可见性**：基于 happens-before——unlock 操作 happens-before 于后续 lock 操作。解锁时刷新工作内存，加锁时从主内存读取
- **有序性**：同步代码块内的指令不允许被重排到块外

注意：synchronized **不保证公平性**，不保证先等待的线程先获取锁。

---

## Q3：synchronized 和 ReentrantLock 的主要区别？

**A**：

| 维度     | synchronized       | ReentrantLock        |
| -------- | ------------------ | -------------------- |
| 可中断   | ❌                 | ✅ lockInterruptibly |
| 公平锁   | ❌ 非公平          | ✅ new ReentrantLock(true) |
| 条件变量 | 单一 wait/notify   | 多个 Condition       |
| 释放锁   | 自动（作用域结束） | 必须 finally unlock   |
| 实现层面 | JVM 层面           | API 层面             |

synchronized 更简单安全（自动释放），ReentrantLock 更灵活强大。JDK 6 之后 synchronized 性能大幅优化，一般优先使用 synchronized。
---
```java
// synchronized 可重入特性
class ReentrantDemo {
    synchronized void methodA() {
        System.out.println("methodA");
        methodB();  // 同一线程可重入，不会死锁
    }
    synchronized void methodB() {
        System.out.println("methodB");
    }
}

// 同步方式
synchronized (lock) { /* 代码块 */ }
synchronized static void method() { /* 类锁 */ }
```


## 关联知识点
