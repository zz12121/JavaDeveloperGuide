---
title: newCondition
tags:
  - Java/并发
  - 问答
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# newCondition

## Q1：Condition 和 Object 的 wait/notify 有什么区别？

**A**：

| 维度 | Object wait/notify | Condition await/signal |
|------|-------------------|----------------------|
| 等待队列 | 每个 Monitor 一个 | 每个 Condition 独立一个 |
| 使用前提 | 必须在 synchronized 内 | 必须在 lock() 内 |
| 等待方法 | wait() | await() |
| 唤醒方法 | notify() / notifyAll() | signal() / signalAll() |
| 超时等待 | wait(timeout) | await(timeout) |
| 不可中断 | 不支持 | awaitUninterruptibly() |
| 精确唤醒 | 只能随机唤醒一个 | 可按 Condition 精确唤醒 |

核心优势：**多个 Condition 实现精确的条件控制**，比如生产者-消费者中 notFull/notEmpty 分别通知。

---

## Q2：await() 方法执行时发生了什么？

**A**：

1. **加入等待队列**：当前线程节点尾插到 Condition 的等待队列
2. **完全释放锁**：调用 `fullyRelease()` 将 state 重置为 0
3. **阻塞等待**：`LockSupport.park()` 阻塞
4. **被唤醒后转移到 CLH 队列**：从 Condition 等待队列转移到 AQS 的 CLH 同步队列
5. **重新竞争锁**：在 CLH 队列中自旋等待获取锁
6. **获取锁后返回**：await() 方法返回

关键：await 期间锁被完全释放，其他线程可以获取锁。

---

## Q3：生产者-消费者模式中如何使用多个 Condition？

**A**：

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();   // 缓冲区不满
Condition notEmpty = lock.newCondition();  // 缓冲区不空

// 生产者
lock.lock();
try {
    while (queue.size() == capacity)
        notFull.await();    // 满了在 notFull 上等待
    queue.add(item);
    notEmpty.signal();      // 唤醒一个在 notEmpty 上等待的消费者
} finally {
    lock.unlock();
}

// 消费者
lock.lock();
try {
    while (queue.isEmpty())
        notEmpty.await();   // 空了在 notEmpty 上等待
    queue.poll();
    notFull.signal();       // 唤醒一个在 notFull 上等待的生产者
} finally {
    lock.unlock();
}
```

对比 Object 方式：只能用 `notifyAll()` 唤醒所有等待者，无法区分生产者和消费者，效率更低。


