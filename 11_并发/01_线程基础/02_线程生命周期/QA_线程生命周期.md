---
title: 线程生命周期
tags:
  - Java/并发
  - 问答
  - 原理型
module: 01_线程基础
created: 2026-04-18
---

# 线程生命周期（NEW → RUNNABLE → BLOCKED → WAITING → TIMED_WAITING → TERMINATED）

## Q1：Java 线程有哪几种状态？

**A**：Java 线程有 6 种状态，定义在 `java.lang.Thread.State` 枚举中：

| 状态 | 说明 |
|------|------|
| **NEW** | 线程创建但未调用 `start()` |
| **RUNNABLE** | 调用 `start()` 后，等待或正在 CPU 上执行 |
| **BLOCKED** | 等待获取 monitor 锁 |
| **WAITING** | 无限等待，需其他线程显式唤醒 |
| **TIMED_WAITING** | 限时等待，超时后自动唤醒 |
| **TERMINATED** | `run()` 执行完毕或抛出异常 |

---

## Q2：BLOCKED 和 WAITING 有什么区别？

**A**：

| 特性 | BLOCKED | WAITING |
|------|---------|---------|
| 等待对象 | monitor 锁 | 条件/通知 |
| 触发方式 | 竞争 synchronized 锁失败 | 主动调用 `wait()`/`join()` |
| 恢复方式 | 锁被释放后自动获取 | 必须被 `notify()`/`notifyAll()` 唤醒 |
| 典型场景 | synchronized 代码块 | 生产者-消费者模式 |

---

## Q3：sleep() 和 wait() 进入的状态一样吗？

**A**：不一样。`sleep()` 进入 **TIMED_WAITING**，不释放锁；`wait()` 进入 **WAITING**（或带超时的 TIMED_WAITING），但会释放锁。两者都可以通过中断提前唤醒。

---

## Q4：yield() 方法的作用？

**A**：`yield()` 表示当前线程愿意让出 CPU 时间片，回到 RUNNABLE 状态重新竞争调度，不释放锁。不保证其他线程一定能获得 CPU。

---

## Q5：join() 方法阻塞哪个线程？

**A**：调用 `join()` 的线程会被阻塞，等待目标线程执行完毕。
---
```java
// 线程状态演示
Thread t = new Thread(() -> {
    // RUNNABLE 状态
    try {
        Thread.sleep(1000);  // TIMED_WAITING
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});
System.out.println(t.getState());  // NEW

t.start();
System.out.println(t.getState());  // RUNNABLE

synchronized (lock) {
    // 如果竞争失败 → BLOCKED
}
// t.join() → 调用线程 WAITING
```


## 关联知识点
