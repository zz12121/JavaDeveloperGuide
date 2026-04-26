
# ScheduledThreadPoolExecutor

## 核心结论

ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor，增加了延迟和周期调度能力。通过 DelayedWorkQueue 和重新入队实现周期执行。

## 深度解析

### 核心：ScheduledFutureTask

```java
private class ScheduledFutureTask<V> extends FutureTask<V> implements RunnableScheduledFuture<V> {
    long time;        // 下次执行时间（纳秒）
    long period;      // 周期（0=一次性，>0=固定速率，<0=固定延迟）
    long sequenceNumber;
    
    public void run() {
        if (!isPeriodic()) {
            ScheduledFutureTask.super.run(); // 一次性执行
        } else if (ScheduledFutureTask.super.runAndReset()) { // 重置状态
            setNextRunTime(); // 计算下次执行时间
            reExecutePeriodic(outerTask); // 重新入队
        }
    }
}
```

### 周期执行原理

```
1. 任务执行完成
2. runAndReset() 重置 FutureTask 状态（不设 result）
3. setNextRunTime() 计算下次执行时间
   - period > 0: time = currentTime + period（固定速率）
   - period < 0: time = currentTime + (-period)（固定延迟）
4. reExecutePeriodic() 将任务重新放入 DelayedWorkQueue
```

### reExecutePeriodic

```java
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        super.getQueue().add(task); // 重新入队
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart(); // 确保至少一个线程处理
    }
}
```

### 继承关系

```
ThreadPoolExecutor
  └── ScheduledThreadPoolExecutor
       - schedule() → 一次性延迟
       - scheduleAtFixedRate() → 固定速率周期
       - scheduleWithFixedDelay() → 固定延迟周期
```

## 易错点与踩坑

### 1. runAndReset() 失败导致周期终止
```java
// ScheduledFutureTask.run() 中：
// 如果 runAndReset() 返回 false → 任务不再重新入队 → 周期结束
// runAndReset() 失败条件：
// 1. 任务已被 cancel
// 2. FutureTask 状态不是 NEW（上一次执行设置了结果或异常）
// 关键：如果任务抛异常 → 状态会被设为 EXCEPTIONAL → runAndReset 返回 false
```

### 2. period = 0 与 period < 0 的区别
- `period = 0`：一次性任务（schedule 方法）
- `period > 0`：固定速率（scheduleAtFixedRate）
- `period < 0`：固定延迟（scheduleWithFixedDelay），实际延迟 = `-period`

### 3. decorateTask 的扩展点
```java
// ScheduledThreadPoolExecutor 提供了 decorateTask 扩展点
// 可以在子类中覆写来定制 ScheduledFutureTask
@Override
protected <V> RunnableScheduledFuture<V> decorateTask(
    Runnable r, RunnableScheduledFuture<V> task) {
    // 自定义扩展
    return task;
}
```

### 4. ensurePrestart 与核心线程
- `ensurePrestart()` 只保证至少一个线程在工作
- 与 `prestartCoreThread` 类似，但不要求 core 线程
- 如果 corePoolSize=0（newSingleThreadScheduledExecutor），也能正常工作
## 关联知识点
