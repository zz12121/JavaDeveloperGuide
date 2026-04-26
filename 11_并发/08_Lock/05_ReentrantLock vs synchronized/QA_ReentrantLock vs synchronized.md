---
title: ReentrantLock vs synchronized
tags:
  - Java/并发
  - 问答
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock vs synchronized

## Q1：ReentrantLock 和 synchronized 有什么区别？

**A**：

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 锁释放 | 自动（退出同步块） | 手动 finally unlock |
| 可中断 | 不可 | lockInterruptibly() |
| 超时 | 不支持 | tryLock(timeout) |
| 公平 | 非公平 | 可选公平/非公平 |
| 条件变量 | 单一 | 多个 Condition |
| tryLock | 不支持 | 支持 |
| 实现层 | JVM 内置 | Java API |

---

## Q2：实际开发中应该选 synchronized 还是 ReentrantLock？

**A**：**优先用 synchronized**。理由：

1. synchronized 语法简洁，自动释放锁，不易出错
2. JDK6 后 JVM 对 synchronized 做了大量优化（偏向锁、轻量级锁），性能差距极小
3. ReentrantLock 需要手动 finally unlock，忘记释放会导致死锁

**仅在需要以下特性时才选 ReentrantLock**：
- 可中断获取锁
- 超时获取锁
- 公平锁
- 多个条件变量
- tryLock 非阻塞尝试
- 锁状态查询

---

## Q3：synchronized 的 JVM 优化有哪些？为什么性能不差？

**A**：JDK6 引入了多种锁优化：

1. **偏向锁**：对象头记录线程 ID，同一线程再次进入无需 CAS
2. **轻量级锁**：CAS 替代互斥量，竞争不激烈时性能高
3. **锁粗化**：多次相邻加锁/解锁合并为一次
4. **锁消除**：逃逸分析确认不会逃逸的锁直接消除

这些优化使得低竞争场景下 synchronized 性能与 ReentrantLock 几乎无异。

---

## Q4：ReentrantLock 在哪些场景下是必须的？

**A**：

1. **实现生产者-消费者模型**：需要多个 Condition 分别控制"不满"和"不空"
2. **定时任务调度**：tryLock(timeout) 避免任务积压
3. **可取消的锁获取**：lockInterruptibly() 响应线程中断
4. **公平调度**：公平锁减少线程饥饿
5. **链式锁获取**：tryLock 非阻塞尝试，避免死锁



> **代码示例：两种锁的典型用法对比**

```java
// synchronized 用法（自动释放）
synchronized (lock) {
    // 临界区
} // 退出时自动释放

// ReentrantLock 用法（手动释放）
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock(); // 必须在 finally 中释放
}

// ReentrantLock 独有特性：超时获取
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try { /* 拿到锁，执行任务 */ }
    finally { lock.unlock(); }
} else {
    // 超时未获取，执行降级逻辑
}
```

