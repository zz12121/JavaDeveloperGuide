
# ScheduledThreadPoolExecutor

## Q1：周期任务的执行原理是什么？

**A**：

1. 任务执行 `run()` → 调用 `runAndReset()`（重置 FutureTask 状态，不设 result）
2. `setNextRunTime()` 计算下次执行时间
3. `reExecutePeriodic()` 将任务重新放入 DelayedWorkQueue
4. 工作线程从队列 take 到期任务执行

如此循环，直到任务被取消或抛异常。

---

## Q2：为什么周期任务要用 runAndReset 而不是 run？

**A**：FutureTask.run() 执行后 state 变为 COMPLETED，不能再执行。

`runAndReset()` 只执行任务但**不设置 result，不更新 state**，保持任务可重新执行。如果任务抛异常，runAndReset 返回 false，不再重新入队。

---

## Q3：ScheduledThreadPoolExecutor 如何保证周期任务不并发？

**A**：每个周期任务只有一个实例，在 DelayedWorkQueue 中排队。工作线程 take 到期任务执行，执行完后重新入队（排在下次到期时间）。同一时刻只有一个线程执行同一个周期任务。

---
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// 延迟 3 秒后执行一次
scheduler.schedule(() -> System.out.println("一次性任务"), 3, TimeUnit.SECONDS);

// 固定间隔 1 秒执行（任务开始后计时）
scheduler.scheduleAtFixedRate(() -> System.out.println("固定间隔"), 0, 1, TimeUnit.SECONDS);

// 固定延迟 1 秒执行（任务结束后计时）
scheduler.scheduleWithFixedDelay(() -> System.out.println("固定延迟"), 0, 1, TimeUnit.SECONDS);
```

