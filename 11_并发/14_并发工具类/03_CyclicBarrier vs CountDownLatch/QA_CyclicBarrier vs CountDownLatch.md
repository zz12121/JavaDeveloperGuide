
# CyclicBarrier vs CountDownLatch

## Q1：CyclicBarrier 和 CountDownLatch 的核心区别？

**A**：

- **CountDownLatch**：一个线程（通常是主线程）等待 N 个其他线程完成操作。事件驱动，`countDown()` 由工作线程调用，`await()` 由等待线程调用。**一次性**。
- **CyclicBarrier**：N 个线程互相等待，全部到达后一起继续。线程对等，每个线程都调用 `await()`。**可重复使用**。

一句话：CountDownLatch 是"**我等你们**"，CyclicBarrier 是"**大家互相等**"。

---

## Q2：它们的底层实现有什么不同？

**A**：

- **CountDownLatch**：基于 **AQS 共享模式**，`state` 作为计数器，`countDown()` CAS 递减 state，到 0 时唤醒等待线程
- **CyclicBarrier**：基于 **ReentrantLock + Condition**，用 `count` 记录剩余等待数，`await()` 获取锁后递减 count，最后一个到达的线程执行 action 并 `signalAll()`

---

## Q3：什么场景选 CountDownLatch，什么场景选 CyclicBarrier？

**A**：

- **选 CountDownLatch**：主线程等 N 个子任务完成（如初始化、并行查询汇总）、一次性使用
- **选 CyclicBarrier**：多线程分阶段协作（如迭代计算、数据分片对齐）、需要重复使用

如果拿不准，想想"谁是等待者"——如果一个明确的线程等其他人，用 CountDownLatch；如果大家都是平等参与者，用 CyclicBarrier。

---

## Q4：能否用 CountDownLatch 替代 CyclicBarrier？

**A**：可以替代"第一轮"，但无法替代"重复使用"的场景。

如果只有一轮等待，两者功能类似。但如果需要多轮同步（如迭代算法每轮对齐），CountDownLatch 无法做到——它是一次性的，count 到 0 后无法重置。

---
```java
// CountDownLatch：一次性，线程等待事件完成
CountDownLatch startLatch = new CountDownLatch(1);
CountDownLatch doneLatch = new CountDownLatch(N);
startLatch.countDown();  // 事件发生
doneLatch.await();       // 等待 N 个任务完成
// 不能重用！

// CyclicBarrier：可重用，线程互相等待
CyclicBarrier barrier = new CyclicBarrier(N);
barrier.await();  // 第 N 个到达时，所有线程同时继续
barrier.await();  // 第二轮
barrier.reset();  // 重置
```


## 关联知识点
