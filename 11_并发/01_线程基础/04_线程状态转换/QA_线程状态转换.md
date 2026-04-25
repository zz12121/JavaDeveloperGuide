---
title: 线程状态转换
tags:
  - Java/并发
  - 问答
  - 原理型
module: 01_线程基础
created: 2026-04-18
---

# 线程状态转换（各种状态之间的转换条件和触发方式）

## Q1：Java 线程的 6 种状态分别是什么？

**A**：定义在 `java.lang.Thread.State` 枚举中：

1. **NEW** — 线程被创建但未调用 `start()`
2. **RUNNABLE** — 调用 `start()` 后，等待或正在 CPU 上执行
3. **BLOCKED** — 等待获取 monitor 锁
4. **WAITING** — 无限等待，等待其他线程唤醒
5. **TIMED_WAITING** — 限时等待，指定时间内等待
6. **TERMINATED** — 线程执行完毕

---

## Q2：各状态之间的转换条件和触发方式？

**A**：

| 转换 | 触发条件 |
|------|----------|
| NEW → RUNNABLE | 调用 `start()` 方法 |
| RUNNABLE → BLOCKED | 尝试获取 synchronized 锁失败 |
| RUNNABLE → WAITING | 调用 `wait()`、`join()`、`LockSupport.park()` |
| RUNNABLE → TIMED_WAITING | 调用 `sleep(time)`、`wait(time)`、`join(time)` |
| BLOCKED → RUNNABLE | 获取到 monitor 锁 |
| WAITING → RUNNABLE | 被 `notify()` / `notifyAll()` 唤醒 |
| TIMED_WAITING → RUNNABLE | 超时自动唤醒或被通知唤醒 |
| RUNNABLE → TERMINATED | `run()` 执行完毕或抛出异常 |

---

## Q3：BLOCKED 和 WAITING 的区别？

**A**：

| 特性 | BLOCKED | WAITING |
|------|---------|---------|
| 等待对象 | monitor 锁 | 条件/通知 |
| 触发方式 | 竞争锁失败 | 主动调用 `wait()`/`join()` |
| 恢复方式 | 锁被释放后自动获取 | 必须被 `notify`/`notifyAll` 唤醒 |
| 典型场景 | synchronized 代码块 | 生产者-消费者模式 |

---

## Q4：RUNNABLE 与操作系统线程状态的关系？

**A**：Java 的 RUNNABLE 状态对应操作系统层面的 **Ready** 和 **Running** 两种状态。线程调用 `yield()` 会从 Running 回到 Ready，但仍属于 RUNNABLE。

---

## Q5：如何查看线程当前状态？

**A**：调用 `Thread.getState()` 方法返回 `Thread.State` 枚举值。还可以通过 `jstack` 命令或 VisualVM 等工具查看 JVM 中所有线程的状态。

---

## Q6：线程调用 start() 两次会怎样？

**A**：抛出 `IllegalThreadStateException`。因为线程状态只能从 NEW → RUNNABLE 转换一次，TERMINATED 状态的线程也不能重新启动。

---

## Q7：为什么 wait() 必须在 synchronized 块内调用？

**A**：`wait()` 方法会释放对象的 monitor 锁，必须先持有锁才能释放。如果在 synchronized 外调用，会抛出 `IllegalMonitorStateException`。

---

## Q8：sleep() 和 wait() 都会让线程暂停，它们的状态一样吗？

**A**：不一样。`sleep()` 进入 TIMED_WAITING，不释放锁；`wait()` 进入 WAITING/TIMED_WAITING，但会释放锁。两者都可以通过中断提前唤醒。

---

## Q9：线程状态转换是原子操作吗？

**A**：不是。状态转换由 JVM 和操作系统共同完成，中间可能经历多个步骤。例如从 BLOCKED 到 RUNNABLE，需要获取锁、修改线程状态、加入调度队列等操作。

```java
// 状态转换演示
Object lock = new Object();
Thread t = new Thread(() -> {
    synchronized (lock) {  // 获取锁：BLOCKED → RUNNABLE
        try {
            lock.wait();   // WAITING（释放锁）
        } catch (InterruptedException e) {
            // 被中断：WAITING → RUNNABLE
        }
    }
});

t.start();                    // NEW → RUNNABLE
t.getState();                 // RUNNABLE
Thread.sleep(50);
// t 在 lock.wait()：WAITING

synchronized (lock) {
    lock.notify();            // 唤醒 t：WAITING → BLOCKED（等待重新获取锁）
}
// t 退出 synchronized：RUNNABLE → TERMINATED
```


## 关联知识点
