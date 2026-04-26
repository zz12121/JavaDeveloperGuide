
# 线程池监控

## Q1：ThreadPoolExecutor 提供了哪些监控方法？

**A**：

| 方法 | 说明 |
|------|------|
| `getActiveCount()` | 活跃线程数 |
| `getPoolSize()` | 当前线程总数 |
| `getQueue().size()` | 队列待处理任务数 |
| `getTaskCount()` | 总任务数 |
| `getCompletedTaskCount()` | 已完成任务数 |
| `getLargestPoolSize()` | 历史最大线程数 |

---

## Q2：如何实现线程池告警？

**A**：

定时任务监控 + 阈值判断：

```java
if (pool.getActiveCount() >= pool.getCorePoolSize() 
    && pool.getQueue().size() > 100) {
    // 队列堆积告警
}

if (pool.getQueue().remainingCapacity() < 10) {
    // 队列即将满，告警
}
```

也可以通过 JMX 暴露线程池指标，接入 Prometheus + Grafana。


