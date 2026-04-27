
# CountDownLatch

## Q1：CountDownLatch 的实现原理是什么？

**A**：基于 AQS 共享模式实现：

1. **初始化**：`CountDownLatch(int count)` 将 `state` 设为 count
2. **await()**：调用 `tryAcquireShared(1)`，如果 `state != 0` 则进入 AQS 等待队列并 park
3. **countDown()**：调用 `tryReleaseShared(1)`，CAS 递减 state，减到 0 时调用 `doReleaseShared()` 唤醒所有等待线程
4. 唤醒后等待线程重新调用 `tryAcquireShared`，此时 `state == 0`，返回成功

本质是**共享锁**：`await` 获取共享锁（`state == 0` 时可获取），`countDown` 释放共享资源。

---

## Q2：CountDownLatch 和 join() 有什么区别？

**A**：

| 对比 | join() | CountDownLatch |
|------|--------|----------------|
| 控制粒度 | 等待特定线程结束 | 等待计数器归零 |
| 使用场景 | 线程对象已知 | 未知数量的任务 |
| 灵活性 | 只能等线程死亡 | 可以由任何线程 countDown |
| 超时 | 不支持 | `await(timeout)` |
| 重用 | 不适用 | 不可重用（一次性的） |

**典型场景**：线程池中提交 N 个任务，用 CountDownLatch 等待所有任务完成，而 join 无法用于线程池。

---

## Q3：countDown() 放在 try 还是 finally 中？

**A**：必须放在 `finally` 中。

```java
try {
    doWork();
} finally {
    latch.countDown(); // 保证异常时也能递减
}
```

如果只放在 try 中，任务抛异常后计数器不会减，await 会永远阻塞。

---

## Q4：CountDownLatch 计数到 0 后还能用吗？

**A**：不能。CountDownLatch 是**一次性**的，count 减到 0 后：

- 所有 await 的线程会被唤醒
- 后续的 await() 调用立即返回（因为 state 已经是 0）
- 但计数器无法重置，无法再次使用

如果需要重复使用，应该用 **CyclicBarrier**。


# CountDownLatch使用场景

## Q1：举例说明 CountDownLatch 的使用场景？

**A**：

1. **并行处理汇总**：主线程提交 N 个查询任务到线程池，用 CountDownLatch 等待全部完成后合并结果返回给前端
2. **应用启动等待**：等待配置中心、数据库、缓存等多个组件全部初始化完成后再开始接收请求
3. **并发测试**：用 startLatch 做发令枪，所有线程同时开始；用 doneLatch 统计完成数
4. **任务分片**：大任务拆分为 N 片并行处理，等待所有分片完成后汇总

---

## Q2：并发测试中为什么需要两个 CountDownLatch？

**A**：
- **startLatch（count=1）**：所有工作线程先 `await`，主线程调用 `countDown()` 作为"发令枪"，保证所有线程同时开始
- **doneLatch（count=N）**：每个工作线程完成后 `countDown()`，主线程 `await` 等待全部完成

如果只用 doneLatch，线程创建和启动有先后顺序，无法保证真正同时执行。

---

## Q3：CountDownLatch 在微服务架构中有什么用？

**A**：典型场景是**服务启动依赖检查**：
- 等待注册中心连接成功
- 等待数据库连接池就绪
- 等待 Redis 连接建立
- 等待消息队列消费者初始化

所有依赖就绪后才开始对外提供服务，避免请求进来时服务还没准备好。

---

```java
// 场景1：并行处理汇总
CountDownLatch latch = new CountDownLatch(3);
ExecutorService pool = Executors.newFixedThreadPool(3);

pool.submit(() -> { queryDB(); latch.countDown(); });
pool.submit(() -> { queryCache(); latch.countDown(); });
pool.submit(() -> { queryRPC(); latch.countDown(); });

latch.await();  // 等待三个任务全部完成
// 合并结果返回

// 场景2：并发测试发令枪
CountDownLatch startLatch = new CountDownLatch(1);
CountDownLatch doneLatch = new CountDownLatch(100);
for (int i = 0; i < 100; i++) {
    new Thread(() -> {
        startLatch.await();  // 等待发令
        doWork();
        doneLatch.countDown();
    }).start();
}
startLatch.countDown();  // 发令！所有线程同时开始
doneLatch.await();       // 等待所有线程完成
```