
# 线程池调优

## Q1：线程池调优的关键步骤？

**A**：

1. 确定任务类型（CPU/IO/混合）
2. 设置合理的线程数（参考 CPU 核数公式）
3. 选择有界队列（控制资源）
4. 设置超时时间（非核心线程回收）
5. 选择合适的拒绝策略
6. **监控 + 压测 + 迭代**

## Q2：任务延迟高怎么调优？

**A**：通常是队列堆积导致的：

1. **缩小队列容量**：让更多任务快速触发非核心线程
2. **增大 maximumPoolSize**：增加处理能力
3. **换 SynchronousQueue**：消除排队延迟
4. **使用 CallerRunsPolicy**：提交线程自动限流

## Q3：生产环境线程池的最佳实践？

**A**：

1. **手动创建** ThreadPoolExecutor，不用 Executors
2. **有界队列**，容量根据业务量估算
3. **自定义线程命名**，方便排查
4. **CallerRunsPolicy** 或自定义拒绝策略
5. **注册 shutdown hook** 优雅关闭
6. **监控告警**：队列长度、活跃线程数
7. **压测验证**：上线前充分测试



> **代码示例：生产环境线程池最佳实践**

```java
// 1. 手动创建，自定义所有参数
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors(),      // corePoolSize
    Runtime.getRuntime().availableProcessors() * 2,   // maximumPoolSize
    60L, TimeUnit.SECONDS,                            // keepAliveTime
    new ArrayBlockingQueue<>(1000),                   // 有界队列
    new ThreadFactoryBuilder().setNameFormat("order-pool-%d").build(), // 自定义命名
    new ThreadPoolExecutor.CallerRunsPolicy()          // 拒绝策略
);

// 2. 注册 shutdown hook 优雅关闭
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    pool.shutdown();   // 停止接受新任务
    try {
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // 超时强制关闭
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
    }
}));
```

