---
title: 可重入锁
tags:
  - Java/并发
  - 问答
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# 可重入锁（state记录重入次数，同线程重复lock不阻塞）

## Q1：什么是可重入锁？为什么需要可重入？

**A**：可重入锁允许同一线程多次获取同一把锁而不会死锁。通过 AQS 的 `state` 记录重入次数：

- `lock()`：state + 1
- `unlock()`：state - 1
- state == 0：锁完全释放

**需要可重入的场景**：
1. 递归方法中加锁
2. 子类 synchronized 方法调用父类 synchronized 方法
3. 一个同步方法调用另一个同步方法

如果不可重入，同一线程第二次获取锁时会死锁。

---

## Q2：ReentrantLock 和 synchronized 都支持可重入吗？

**A**：**都支持**。

- `ReentrantLock`：通过 AQS state 记录重入次数
- `synchronized`：JVM 通过对象头的锁记录（重入计数）支持

```java
// synchronized 可重入
synchronized (obj) {
    synchronized (obj) { // 不会死锁
        // ...
    }
}

// ReentrantLock 可重入
lock.lock();
lock.lock(); // 不会死锁
lock.unlock();
lock.unlock();
```

---

## Q3：lock 和 unlock 次数不匹配会怎样？

**A**：

- **lock 多于 unlock**：锁未释放，其他线程永远等待（死锁）
- **unlock 多于 lock**：抛 `IllegalMonitorStateException`（不是锁持有者）

最佳实践：lock/unlock 严格配对，始终在 `finally` 中 unlock。

---

## Q4：synchronized 的底层原理是什么？

**A**：`synchronized` 由 JVM 实现，编译后生成 `monitorenter`/`monitorexit` 字节码指令：

```java
// Java 源码
synchronized (obj) {
    // ...
}

// 编译后的字节码
monitorenter obj          // 获取锁
// ...
monitorexit obj           // 释放锁
```

JVM 底层使用 `ObjectMonitor`（C++ 实现）管理锁：

```cpp
class ObjectMonitor {
    Thread* _owner;        // 持有者线程
    int _recursions;       // 重入次数
    ObjectWaiter* _WaitSet; // 等待队列（wait() 的线程）
    int _count;            // 等待者数量
};
```

线程获取 monitor 失败时会被 park，进入内核态等待——这就是 synchronized "重量级锁" 慢的原因。

---

## Q5：synchronized 的锁升级机制是怎样的？

**A**：JVM 为 synchronized 实现了四种锁状态，逐级升级（不可降级）：

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
       ↓           ↓           ↓
    Mark Word   CAS 栈帧    ObjectMonitor
    存线程ID    无竞争时    激烈竞争时
```

**偏向锁**（无竞争）：Mark Word 记录线程ID，第一次获取后不再需要 CAS，直接检查线程ID即可。

**轻量级锁**（少量竞争）：Mark Word 复制到线程栈帧的 Lock Record，原位置 CAS 指向该记录。线程在用户态自旋等待。

**重量级锁**（激烈竞争）：Mark Word 指向堆中的 ObjectMonitor，等待线程被 park，进入内核态等待，性能最差。

```java
// 关闭偏向锁延迟（测试用）
-XX:BiasedLockingStartupDelay=0

// 禁用偏向锁
-XX:-UseBiasedLocking

// JDK 15+ 偏向锁已废弃
--add-opens=java.base/java.lang=ALL-UNNAMED
```

---

## Q6：为什么说 synchronized 是非公平的？

**A**：`synchronized` 不保证等待线程的 FIFO 顺序。具体原因：

1. **monitorenter 的实现**：线程通过 OS 的互斥量争抢，没有排队机制
2. **唤醒顺序不确定**：被唤醒的线程和其他新到达的线程竞争
3. **不存在等待队列**：不像 AQS 有 CLH 队列严格维护顺序

```java
// synchronized 的唤醒行为（不公平）：
// T1（持有锁）→ T2/T3 入队等待 → T4 新线程到达
// T1 释放 → T2 刚被唤醒 → T4 同时 CAS 抢锁
// 结果：T4 可能先于 T2 获取锁（非 FIFO）

// ReentrantLock 公平锁的保证：
// T1 释放 → hasQueuedPredecessors() 检查
// → 队列有 T2 → T2 一定先获取（队列保证了 FIFO）
```

