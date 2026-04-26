
# prestartCoreThread

## Q1：为什么线程池默认不预热核心线程？

**A**：ThreadPoolExecutor 默认懒加载，核心线程在首次提交任务时才创建（`execute` → `addWorker`）。

这样设计是为了避免资源浪费：如果线程池创建后很长时间没有任务，不预热可以节省线程资源。

## Q2：什么时候需要预热核心线程？

**A**：需要**低延迟响应**的场景：

- 交易系统：首个请求零延迟
- 网关服务：避免请求"冷启动"
- 定时任务：任务触发时线程已就绪

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(...);
pool.prestartAllCoreThreads(); // 启动时预热
```

