
# newScheduledThreadPool

## Q1：scheduleAtFixedRate 和 scheduleWithFixedDelay 有什么区别？

**A**：

- **scheduleAtFixedRate**：固定**开始时间**间隔（从上次开始算）
  - 任务 1s 完成，间隔 2s → 每 2s 执行
  - 任务 3s 完成，间隔 2s → 上次结束立即执行（补执行）
  
- **scheduleWithFixedDelay**：固定**结束时间**间隔（从上次结束算）
  - 无论任务耗时多久，上一次结束后等固定时间再执行

---

## Q2：周期任务抛异常会怎样？

**A**：**后续周期任务不再执行**。

ScheduledThreadPoolExecutor 中，周期任务的异常会导致后续调度被取消。所以周期任务必须 try-catch：

```java
scheduler.scheduleAtFixedRate(() -> {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("周期任务异常", e);
    }
}, 0, 1, TimeUnit.SECONDS);
```

---

## Q3：ScheduledThreadPoolExecutor 使用什么队列？

**A**：`DelayedWorkQueue`，基于 PriorityQueue（二叉堆）按执行时间排序。无界队列，任务过多可能 OOM。

