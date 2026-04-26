---
title: Lock vs synchronized
tags:
  - Java/并发
  - 问答
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# Lock vs synchronized（可中断/可超时/公平/非公平/多个条件变量/tryLock尝试获取）

## Q1：Lock 和 synchronized 有什么区别？

**A**：核心区别：

| 维度 | synchronized | Lock |
|------|-------------|------|
| 获取方式 | 隐式 | 显式（lock/unlock） |
| 可中断 | ❌ | ✅ lockInterruptibly() |
| 可超时 | ❌ | ✅ tryLock(timeout) |
| 非阻塞尝试 | ❌ | ✅ tryLock() |
| 公平锁 | ❌ | ✅ ReentrantLock(true) |
| 条件变量 | 单一 wait set | 多个 Condition |
| 异常自动释放 | ✅ | ❌ 必须 finally |
| 底层 | JVM 对象头+Monitor | Java API (AQS+CAS) |

---

## Q2：什么时候用 Lock，什么时候用 synchronized？

**A**：

**用 synchronized**（优先选择）：
- 代码简单，不需要 Lock 额外功能
- 竞争不激烈
- 团队熟悉，代码可读性好

**用 Lock**：
- 需要**可中断/可超时**获取锁
- 需要**公平锁**
- 需要**多个条件变量**（如 BlockingQueue 的 notFull/notEmpty）
- 需要**尝试获取锁**避免死锁
- 实现**自定义同步器**

---

## Q3：Lock 的多条件变量有什么优势？

**A**：synchronized 只有一个等待队列（wait set），notify/notifyAll 无法精确选择唤醒哪类线程。Lock 可以创建多个 Condition，实现**精确唤醒**：

```java
// 生产者-消费者
Condition notFull = lock.newCondition();  // 生产者等待
Condition notEmpty = lock.newCondition(); // 消费者等待

// 生产者放入后只唤醒消费者
notEmpty.signal();  // 精确唤醒

// synchronized 的 notifyAll 会唤醒所有线程（包括其他生产者）
notifyAll();  // 不精确，浪费唤醒
```


