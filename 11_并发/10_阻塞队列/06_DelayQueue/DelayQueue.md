---
title: DelayQueue
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# DelayQueue

## 核心结论

DelayQueue 是**无界延迟阻塞队列**，元素必须实现 Delayed 接口。只有**过期**（delay <= 0）的元素才能出队。底层基于 PriorityBlockingQueue。

## 深度解析

### Delayed 接口

```java
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit); // 返回剩余延迟时间
}

// 典型实现
class DelayedElement implements Delayed {
    private final long expireTime;
    
    DelayedElement(long delay, TimeUnit unit) {
        this.expireTime = System.currentTimeMillis() + unit.toMillis(delay);
    }
    
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(expireTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }
    
    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.expireTime, ((DelayedElement)o).expireTime);
    }
}
```

### 原理

```java
public class DelayQueue<E extends Delayed> implements BlockingQueue<E> {
    private final PriorityQueue<E> q = new PriorityQueue<>(); // 按过期时间排序
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
}
```

- **PriorityQueue** 按 getDelay 排序，最先过期的排在堆顶
- take 时检查堆顶元素是否过期，未过期则 await 直到过期

### take 源码

```java
public E take() throws InterruptedException {
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) available.await(); // 队列空，等待
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0) return q.poll(); // 已过期，出队
                available.awaitNanos(delay); // 未过期，等待剩余时间
            }
        }
    } finally {
        lock.unlock();
    }
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 有界性 | 无界 |
| 出队条件 | delay <= 0（已过期） |
| 排序 | 按过期时间 |
| 元素要求 | 必须实现 Delayed |
| put | 不阻塞（无界） |
| take | 未过期时阻塞等待 |

### 典型应用

```java
// 1. 延迟任务
DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask(5000, () -> System.out.println("5秒后执行")));

// 2. 缓存过期
DelayQueue<DelayedCacheEntry> expiredQueue = new DelayQueue<>();
// 过期元素出队后清理缓存

// 3. 定时器
class DelayedTimerTask implements Delayed { ... }
```

### 与 ScheduledThreadPoolExecutor 的关系

`ScheduledThreadPoolExecutor` 内部使用 `DelayedWorkQueue`（DelayQueue 的定制变体）实现延迟任务调度：

```java
// ScheduledThreadPoolExecutor 构造时创建 DelayedWorkQueue
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());  // 基于 DelayQueue 实现
}
```

**DelayedWorkQueue vs DelayQueue**：

| 维度 | DelayQueue | DelayedWorkQueue |
|------|------------|------------------|
| 底层 | PriorityQueue | 自定义堆（RunnableScheduledFuture[]） |
| 元素类型 | 任意 Delayed 实现 | RunnableScheduledFuture（任务包装器） |
| 用途 | 通用延迟队列 | ScheduledThreadPoolExecutor 专用 |
| 取消机制 | 无 | 支持任务取消（从堆中移除） |
| 线程安全 | ReentrantLock | ReentrantLock |

**核心关联**：`schedule()` 提交的任务被包装为 `ScheduledFutureTask`（实现 RunnableScheduledFuture → Delayed），放入 DelayedWorkQueue，按延迟时间排序，到期后由线程池执行。

## 关联知识点
