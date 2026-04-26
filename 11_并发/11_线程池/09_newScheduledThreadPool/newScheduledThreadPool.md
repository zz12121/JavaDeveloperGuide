
# newScheduledThreadPool

## 核心结论

定时/周期线程池，使用 DelayedWorkQueue 存储任务。支持延迟执行和周期执行。

## 深度解析

### 创建参数

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

// ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

### 三种调度方式

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(3);

// 1. 延迟执行（一次性）
scheduler.schedule(() -> System.out.println("3秒后执行"), 3, TimeUnit.SECONDS);

// 2. 固定速率（从上次开始时间算间隔）
scheduler.scheduleAtFixedRate(
    () -> System.out.println("固定速率"),
    1, 2, TimeUnit.SECONDS // 初始延迟1秒，之后每2秒
);
// 如果任务耗时1秒 → 每2秒执行
// 如果任务耗时3秒 → 任务结束后立即执行下一次（不会并发执行）

// 3. 固定延迟（从上次结束时间算间隔）
scheduler.scheduleWithFixedDelay(
    () -> System.out.println("固定延迟"),
    1, 2, TimeUnit.SECONDS // 初始延迟1秒，上一次结束后等2秒
);
```

### scheduleAtFixedRate vs scheduleWithFixedDelay

```
scheduleAtFixedRate(task, 1, 2, s):
  开始时间: 1s, 3s, 5s, 7s...（固定间隔，不管任务耗时）
  如果任务超过2秒 → 上次结束立即开始下一次

scheduleWithFixedDelay(task, 1, 2, s):
  开始时间: 1s, 1+耗时+2s, ...（上一次结束后再等2秒）
  保证两次执行之间至少间隔2秒
```

### DelayedWorkQueue

- 基于 PriorityQueue（二叉堆），按执行时间排序
- 无界队列，任务过多可能 OOM
- take 时检查堆顶任务是否到期

## 易错点与踩坑

### 1. 周期任务异常导致后续任务停止
```java
// ❌ 如果任务抛异常，后续周期调度自动停止
scheduler.scheduleAtFixedRate(() -> {
    int i = 1 / 0; // ArithmeticException
    System.out.println("执行");
}, 0, 1, SECONDS);
// 只执行一次就停了，后续不再调度

// ✅ 任务内部 try-catch
scheduler.scheduleAtFixedRate(() -> {
    try {
        int i = 1 / 0;
    } catch (Exception e) {
        log.error("周期任务异常", e);
    }
}, 0, 1, SECONDS);
```

### 2. scheduleAtFixedRate 任务堆积
```java
// 如果任务执行时间 > period，不会并发执行
// 而是"追赶"：上一次结束立即开始下一次
scheduler.scheduleAtFixedRate(() -> {
    Thread.sleep(3000); // 任务耗时 3 秒，period = 2 秒
    System.out.println("执行时间: " + System.currentTimeMillis());
}, 0, 2, SECONDS);
// 实际执行间隔 ≈ 3 秒（而非 2 秒）
// 大量堆积会导致任务密集执行，类似"补偿"
```

### 3. DelayedWorkQueue 无界导致 OOM
- 与其他 Executors 工厂方法一样，DelayedWorkQueue 是无界的
- 如果任务提交速度 > 执行速度 → 任务堆积 → OOM

### 4. shutdown 后周期任务不执行
```java
scheduler.shutdown();
// 已提交的周期任务不会继续执行
// 队列中等待的周期任务被丢弃
```
## 关联知识点
