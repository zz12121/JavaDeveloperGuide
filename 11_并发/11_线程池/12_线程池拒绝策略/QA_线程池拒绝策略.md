
# 线程池拒绝策略

## Q1：JDK 提供了哪些拒绝策略？

**A**：

1. **AbortPolicy**（默认）：抛 RejectedExecutionException
2. **CallerRunsPolicy**：调用者线程执行
3. **DiscardPolicy**：静默丢弃
4. **DiscardOldestPolicy**：丢弃最旧任务，重新提交

---

## Q2：生产环境推荐哪种拒绝策略？

**A**：**CallerRunsPolicy** 或**自定义策略**。

- CallerRunsPolicy：不丢弃任务，由提交线程执行，自带"限流"效果（提交线程被阻塞时无法继续提交）
- 自定义：记录日志、告警、降级处理

AbortPolicy 会抛异常可能导致调用链中断，DiscardPolicy 丢失任务不可接受。

---

## Q3：什么情况下会触发拒绝策略？

**A**：同时满足三个条件：

1. 线程池状态不是 RUNNING
2. workQueue 已满（offer 返回 false）
3. 当前线程数 ≥ maximumPoolSize

**使用无界队列时永远不会触发**，这也是 Executors 创建的线程池有 OOM 风险但不会触发拒绝策略的原因。



> **代码示例：四种拒绝策略行为对比**

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    1, 1, 0L, SECONDS, new ArrayBlockingQueue<>(1));

// 1. AbortPolicy（默认）：抛异常
pool.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
// pool.submit(() -> {}); // RejectedExecutionException

// 2. CallerRunsPolicy：调用者线程执行
pool.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
// 提交线程自己执行被拒绝的任务

// 3. DiscardPolicy：静默丢弃
pool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
// 什么都没发生，任务消失

// 4. DiscardOldestPolicy：丢弃队列头部最旧任务
pool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());
// 队列首任务被丢弃，重新提交当前任务
```


# 拒绝策略使用场景

## Q1：不同业务场景应该选哪种拒绝策略？

**A**：

- **订单/支付**：CallerRunsPolicy（不能丢，自动降速）
- **日志/监控**：DiscardPolicy（丢几条无所谓）
- **实时推送**：DiscardOldestPolicy（保留最新）
- **关键计算**：自定义策略（记录 + 告警 + 降级）

核心原则：**不能丢任务的场景用 CallerRunsPolicy**。

---

## Q2：CallerRunsPolicy 为什么有"限流"效果？

**A**：线程池过载时，提交任务的线程被拉去执行任务，阻塞了提交操作。提交线程忙于执行任务时无法继续提交新任务，自然形成了负反馈限流。

这比抛异常后重试更优雅，既不丢任务也不需要额外限流机制。

> **代码示例：自定义拒绝策略（记录日志 + 告警）**

```java
// 生产环境常用：记录 + 告警 + 降级
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    10, 20, 60L, SECONDS, new ArrayBlockingQueue<>(500));

pool.setRejectedExecutionHandler((r, executor) -> {
    // 1. 记录日志
    log.warn("线程池已满！active={}, queue={}, total={}",
        executor.getActiveCount(), executor.getQueue().size(), executor.getTaskCount());
    // 2. 告警
    alertService.send("线程池过载");
    // 3. 降级处理（放入本地队列或同步执行）
    fallbackQueue.offer(r);
});

// 或直接使用 CallerRunsPolicy 实现自动限流
pool.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

# 自定义拒绝策略

## Q1：线程池有哪些内置拒绝策略？

**A**：

|策略|行为|特点|
|---|---|---|
|`AbortPolicy`|抛出 RejectedExecutionException|**默认策略**，需要调用方感知|
|`CallerRunsPolicy`|调用者线程执行任务|自动降级，阻塞提交线程|
|`DiscardPolicy`|静默丢弃|无感知，可能丢数据|
|`DiscardOldestPolicy`|丢弃队列最旧任务|关注最新数据|

---

## Q2：如何自定义拒绝策略？给出示例

**A**：实现 `RejectedExecutionHandler` 接口：

```java
public class LogAndFallbackHandler implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        log.warn("任务被拒绝, pool={}, queue={}, active={}",
            executor.getPoolSize(),
            executor.getQueue().size(),
            executor.getActiveCount());
        // 降级：发送到MQ或本地队列
        mqTemplate.send("fallback-queue", r);
    }
}

// 使用
new ThreadPoolExecutor(4, 8, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000),
    new NamedThreadFactory("biz"),
    new LogAndFallbackHandler());
```

---

## Q3：生产环境中推荐哪种拒绝策略？

**A**：
1. **首选自定义策略**：日志记录 + 降级（存入 MQ/本地队列）
2. **简单场景用 CallerRunsPolicy**：自动降级，由调用线程执行，天然限流
3. **不推荐 DiscardPolicy**：静默丢弃任务，排查问题困难
4. **AbortPolicy 配合 try-catch**：需要调用方主动处理异常

核心原则：**拒绝不等于丢弃，必须有兜底方案**。

