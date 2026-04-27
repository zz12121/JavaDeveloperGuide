
# CyclicBarrier

## Q1：CyclicBarrier 的实现原理是什么？

**A**：基于 **ReentrantLock + Condition** 实现，而非 AQS 共享模式：

1. 内部维护 `parties`（总线程数）、`count`（剩余等待数）、`lock`（ReentrantLock）、`trip`（Condition）
2. `await()`：获取 lock → count 递减 → 如果 count > 0，调用 `trip.await()` 进入 Condition 等待队列；如果 count == 0，先执行 barrierAction，然后 `trip.signalAll()` 唤醒所有等待线程
3. **generation 对象**标记当前"代"，`reset()` 或屏障破坏时创建新一代
4. 唤醒后 count 重置为 parties，可以再次使用

---

## Q2：CyclicBarrier 为什么可以重复使用？

**A**：因为每次屏障打开后，`count` 会重置为 `parties`，generation 更新为新一代。等待的线程在新一代中重新 await，不会被旧的状态干扰。

内部使用 `generation` 对象（仅含 `broken` 标志），每次 barrier trip 后创建新的 generation，确保上轮的等待线程不受影响。

---

## Q3：什么时候会抛 BrokenBarrierException？

**A**：

1. **等待线程被中断**：其他等待线程会被唤醒并抛 BrokenBarrierException
2. **等待超时**：超时的线程抛 TimeoutException，其他线程抛 BrokenBarrierException
3. **调用 reset()**：重置屏障，正在等待的线程抛 BrokenBarrierException
4. **barrierAction 抛异常**：屏障被破坏

屏障 broken 后，后续的 await() 也会直接抛 BrokenBarrierException，直到 reset()。

---

## Q4：CyclicBarrier 的回调 action 在哪个线程执行？

**A**：在**最后一个到达屏障的线程**中执行。

如果 action 耗时很长，会阻塞最后一个线程。如果需要异步执行，可以在 action 中提交到线程池。

---

## Q5：CyclicBarrier 适用于什么场景？

**A**：

1. **多轮迭代计算**：如矩阵乘法，每轮所有线程完成后才能进入下一轮
2. **多线程数据分片对齐**：每批数据所有分片处理完成后统一汇总
3. **游戏同步**：等待所有玩家就绪后开始
4. **并发测试**：多轮压测，每轮等待所有请求完成

---
```java
// CyclicBarrier：多线程到达同一屏障点后同时继续
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程就绪，开始新一轮");
});

for (int i = 0; i < 3; i++) {
    final int threadId = i;
    new Thread(() -> {
        System.out.println("线程" + threadId + "准备完毕，等待其他线程");
        try {
            barrier.await();  // 等待其他 2 个线程
            System.out.println("线程" + threadId + "开始执行");
        } catch (Exception e) { }
    }).start();
}
// 可重复使用（CountDownLatch 不行）
```

