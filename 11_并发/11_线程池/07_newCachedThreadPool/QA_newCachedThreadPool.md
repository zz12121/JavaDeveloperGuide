
# newCachedThreadPool

## Q1：newCachedThreadPool 的特点是什么？

**A**：

- 核心 0，最大 Integer.MAX_VALUE
- 60 秒空闲回收
- SynchronousQueue（不存储）
- 来一个任务创建一个线程（有空闲则复用）
- 高并发时线程数爆炸

---

## Q2：CachedThreadPool 适合什么场景？

**A**：适合**短生命周期、大量并发**的任务：

- HTTP 请求处理
- 异步日志
- 短时间突发任务

不适合：长时间运行的任务、需要控制并发数量的场景。

---

## Q3：CachedThreadPool 为什么会线程爆炸？

**A**：核心线程 0 + SynchronousQueue 不缓冲 → 每个任务都必须立即有线程接收 → 没有空闲线程就创建新线程。如果任务提交速度 > 任务处理速度，线程数持续增长，直到 Integer.MAX_VALUE 或 OOM。



> **代码示例：CachedThreadPool 线程爆炸演示**

```java
ExecutorService pool = Executors.newCachedThreadPool();
// 等价于：new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>());

// 模拟高并发：任务执行慢于提交速度 → 线程数暴增
for (int i = 0; i < 10000; i++) {
    pool.execute(() -> {
        try { Thread.sleep(5000); } catch (e) {}
    });
}
// 10000个线程同时创建，可能 OOM！

// 生产环境推荐：手动创建有界线程池
ExecutorService safe = new ThreadPoolExecutor(
    10, 50, 60L, SECONDS, new ArrayBlockingQueue<>(1000),
    new ThreadFactoryBuilder().setNameFormat("biz-pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

