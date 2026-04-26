---
title: DelayQueue
tags:
  - Java/并发
  - 问答
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# DelayQueue

## Q1：DelayQueue 的工作原理？

**A**：

1. 元素必须实现 `Delayed` 接口（`getDelay()` + `Comparable`）
2. 底层用 `PriorityQueue` 按**过期时间排序**，最先过期的在堆顶
3. `take()` 时检查堆顶元素：
   - 已过期（delay <= 0）→ 直接出队
   - 未过期 → `awaitNanos(delay)` 等待到过期时间
4. `put()` 不阻塞（无界队列）

---

## Q2：DelayQueue 的典型使用场景？

**A**：

1. **延迟任务**：定时执行任务（如订单超时取消）
2. **缓存过期**：元素到期后从队列取出，清理缓存
3. **心跳检测**：检测连接超时
4. **任务调度**：按优先级和延迟时间调度

---

## Q3：DelayQueue 和 ScheduledThreadPoolExecutor 有什么关系？

**A**：`ScheduledThreadPoolExecutor` 内部使用 `DelayedWorkQueue`（DelayQueue 的定制变体）实现延迟任务调度：

```java
// ScheduledThreadPoolExecutor 构造
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());  // 基于 DelayQueue 实现
}
```

- `schedule()` 提交的任务被包装为 `ScheduledFutureTask`（实现 Delayed）
- 任务按延迟时间排序放入 DelayedWorkQueue
- 到期后由线程池执行

**DelayedWorkQueue 与 DelayQueue 的区别**：DelayedWorkQueue 是 ScheduledThreadPoolExecutor 的专用实现，支持任务取消，底层是 RunnableScheduledFuture[] 而非 PriorityQueue。

---

## Q4：DelayQueue 会有"假唤醒"问题吗？

**A**：DelayQueue 的 take 用 `awaitNanos(delay)` 而非无参 `await()`。唤醒后重新检查 `getDelay()`，如果还没过期继续等待。所以假唤醒不会导致未过期元素被取出。

---
```java
// DelayQueue：延迟队列（订单超时取消）
class DelayedTask implements Delayed {
    private final long expireTime;
    private final String name;

    DelayedTask(String name, long delayMs) {
        this.name = name;
        this.expireTime = System.currentTimeMillis() + delayMs;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(expireTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(getDelay(TimeUnit.MILLISECONDS), o.getDelay(TimeUnit.MILLISECONDS));
    }
}

DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask("取消订单", 30000));  // 30 秒后可取出
DelayedTask task = queue.take();  // 阻塞直到到期
```


## 关联知识点

