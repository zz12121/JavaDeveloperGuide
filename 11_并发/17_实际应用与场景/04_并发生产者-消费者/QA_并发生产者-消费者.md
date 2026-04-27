---
tags: [Java并发, 生产者消费者, BlockingQueue, TransferQueue, 背压]
module: 17_实际应用与场景
chapter: 04_并发生产者-消费者
---

# 并发生产者-消费者

## Q1：怎么用 BlockingQueue 实现生产者-消费者？

**A**：

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);

// 生产者
queue.put(item);         // 队列满时自动阻塞
queue.offer(item, 100, TimeUnit.MILLISECONDS);  // 超时放弃

// 消费者
String item = queue.take();    // 队列空时自动阻塞
String item = queue.poll(100, TimeUnit.MILLISECONDS);  // 超时返回 null
```

`BlockingQueue` 的 `put()` 和 `take()` 自动处理阻塞/唤醒，无需手动 `wait/notify`。

---

## Q2：生产者和消费者速度不匹配怎么办？

**A**：

1. **有界队列**：使用 `ArrayBlockingQueue`，队列满时生产者阻塞，防止 OOM
2. **非阻塞 + 拒绝**：使用 `offer()` 代替 `put()`，满时返回 false，执行降级策略
3. **多消费者**：增加消费者线程数，加快消费速度
4. **背压（Backpressure）**：消费方通知生产方降低速度（如 Reactor 的 `Backpressure`、RxJava 的 `onBackpressureDrop`）

```java
// 背压实现（简单版）：队列水位控制
public void produce(Task task) {
    if (queue.size() > THRESHOLD) {      // 队列超过阈值
        Thread.sleep(100);               // 主动降速（背压）
    }
    queue.put(task);
}
```

---

## Q3：如何优雅停止生产者-消费者？

**A**：

推荐**毒丸模式**（Poison Pill）：

```java
// 定义毒丸对象
static final Task POISON = new Task("POISON");

// 生产者结束时放入毒丸
queue.put(POISON);

// 消费者检测到毒丸后退出
while (true) {
    Task task = queue.take();
    if (task == POISON) break;
    process(task);
}
```

多消费者时，需要放入与消费者数量相同的毒丸（每个消费者消费一个后退出）。

也可以用 `Thread.interrupt()` 中断消费者线程，消费者在 `take()` 抛出 `InterruptedException` 时退出。

---

## Q4：BlockingQueue 各实现怎么选择？

**A**：

| 队列 | 有界 | 特点 | 适用场景 |
|------|------|------|---------|
| `ArrayBlockingQueue` | ✅ | 数组、FIFO、公平/非公平锁 | 内存严格受限，吞吐量中等 |
| `LinkedBlockingQueue` | 可选 | 链表，生产/消费独立锁，吞吐量高 | 通用场景，FixedThreadPool 默认 |
| `PriorityBlockingQueue` | ❌ 无界 | 优先级排序，堆实现 | 任务有优先级 |
| `SynchronousQueue` | 0 | 无缓冲，直接移交 | 生产消费速度匹配，CachedThreadPool 默认 |
| `DelayQueue` | ❌ 无界 | 按延迟时间排序 | 定时任务、超时处理 |
| `LinkedTransferQueue` | ❌ 无界 | 支持直接移交+缓冲，吞吐量最高 | 高性能场景 |

**选型口诀**：
- 内存有限 → `ArrayBlockingQueue`
- 最高吞吐 → `LinkedTransferQueue`
- 任务有优先级 → `PriorityBlockingQueue`
- 生产消费完全同步 → `SynchronousQueue`
- 定时延迟 → `DelayQueue`

---

## Q5：TransferQueue 和 SynchronousQueue 有什么区别？

**A**：

`LinkedTransferQueue` 实现 `TransferQueue`，是 `SynchronousQueue` 的超集：

```java
LinkedTransferQueue<Task> queue = new LinkedTransferQueue<>();

// transfer()：等待消费者来取（无消费者则阻塞）—— 同 SynchronousQueue.put
queue.transfer(task);

// tryTransfer()：立即尝试移交（无消费者立即失败）
boolean ok = queue.tryTransfer(task);

// put()：可以正常入队（不要求消费者立即取走）—— SynchronousQueue 做不到！
queue.put(task);  // 生产者不等消费者，先入队
```

| 维度 | SynchronousQueue | LinkedTransferQueue |
|------|-----------------|-------------------|
| 容量 | 0（必须同步移交） | 无界（支持缓冲） |
| 直接移交 | ✅ | ✅（transfer/tryTransfer） |
| 缓冲入队 | ❌ | ✅（put） |
| 吞吐量 | 中 | 高（更灵活的生产者策略） |

**使用场景**：生产者明确需要确认任务已被某个消费者接收（如异步请求-响应，生产者等消费者回执）。

---

## Q6：什么时候用 Disruptor 代替 BlockingQueue？

**A**：

当 `BlockingQueue` 性能无法满足需求时，考虑 **Disruptor**：

**适用 Disruptor 的场景**：
- 吞吐量要求极高（百万级 TPS）
- 低延迟场景（金融交易、撮合引擎）
- 日志框架（Log4j2 AsyncLogger 内部使用 Disruptor）

**不适用 Disruptor 的场景**：
- 普通业务场景（BlockingQueue 已够用，Disruptor 初始化开销大）
- 需要优先级队列（Disruptor 是严格 FIFO）
- 队列大小需要动态扩展（RingBuffer 固定大小）

```java
// 性能数量级对比（参考 LMAX 论文）
// ArrayBlockingQueue：约 4~5 百万 ops/sec
// Disruptor：约 6 千万+ ops/sec（快约 10 倍）

// 简单判断标准：
// QPS < 100万 → BlockingQueue 够用
// QPS > 100万 → 考虑 Disruptor
```
