
# ThreadPoolExecutor

## Q1：Worker 为什么继承 AQS 且实现不可重入锁？

**A**：

Worker 继承 AQS 并实现**不可重入的独占锁**：
- 执行任务时 state=1（加锁）
- 等待任务时 state=0（解锁）
- 不可重入：shutdown 时调用 `interruptIdleWorkers()`，可以尝试获取 Worker 锁来中断空闲线程

如果是可重入锁，线程在执行任务时（已持有锁）调 shutdown，interrupt 会失败。

---

## Q2：为什么推荐手动创建 ThreadPoolExecutor？

**A**：

Executors 工厂方法的问题：
- **FixedThreadPool / SingleThreadExecutor**：无界队列 → OOM
- **CachedThreadPool**：最大线程 Integer.MAX_VALUE → 线程爆炸
- 没有自定义命名、拒绝策略、线程工厂的能力

手动创建可以精确控制所有参数，避免生产事故。

---

## Q3：ThreadPoolExecutor 提供了哪些钩子方法？

**A**：

- `beforeExecute(Thread, Runnable)`：任务执行前
- `afterExecute(Runnable, Throwable)`：任务执行后（含异常）
- `terminated()`：线程池终止时

用途：监控任务执行时间、统计指标、记录日志、资源清理等。

---
```java
// 手动创建 ThreadPoolExecutor — 生产推荐方式
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize：核心线程数
    8,                              // maximumPoolSize：最大线程数
    60L, TimeUnit.SECONDS,          // keepAliveTime：非核心线程空闲回收时间
    new ArrayBlockingQueue<>(1000),  // workQueue：有界队列
    new ThreadFactory() {           // threadFactory：自定义线程命名
        private final AtomicInteger count = new AtomicInteger(0);
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "biz-thread-" + count.getAndIncrement());
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // handler：拒绝策略
);
```

```java
// 钩子方法示例 — 监控任务执行时间
ThreadPoolExecutor monitor = new ThreadPoolExecutor(4, 8, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000)) {
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        ((CustomTask) r).setStartTime(System.nanoTime());
    }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        long elapsed = System.nanoTime() - ((CustomTask) r).getStartTime();
        if (elapsed > 1_000_000) {  // 超过 1ms 告警
            log.warn("Task slow: {}ms", elapsed / 1_000_000);
        }
    }
};
```

