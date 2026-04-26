---
title: 生产者-消费者模式
tags:
  - Java/并发
  - 问答
  - 场景型
module: 10_阻塞队列
created: 2026-04-18
---

# 生产者-消费者模式

## Q1：生产者-消费者模式的核心思想是什么？

**A**：通过**阻塞队列**解耦生产者和消费者：

- **生产者**：将数据 put 到队列，队列满时自动阻塞
- **消费者**：从队列 take 数据，队列空时自动阻塞
- **队列**：缓冲数据，平衡生产消费速度

不需要手动加锁、notify/notifyAll，BlockingQueue 自动处理同步。

---

## Q2：如何优雅地停止消费者线程？

**A**：

1. **毒丸模式**：生产者结束后放入特殊标记
```java
final String POISON = "STOP";
queue.put(POISON); // 消费者收到后退出
```

2. **中断机制**：调用 consumer.interrupt()，take 抛 InterruptedException
3. **CompletableFuture**：用 Future 管理消费者生命周期
4. **ExecutorService.shutdown()**：线程池统一关闭

---

## Q3：生产者-消费者模式有什么优势？

**A**：

1. **解耦**：生产者和消费者不直接交互
2. **削峰**：突发流量被队列缓冲
3. **异步**：生产者不等待消费者处理完
4. **可扩展**：增减生产者/消费者数量无需修改代码
5. **线程安全**：BlockingQueue 保证数据安全

---

## Q4：线程池应该选择哪种阻塞队列？

**A**：不同线程池使用不同队列，选型影响系统稳定性：

| 线程池 | 默认队列 | 特点 | 风险 |
|--------|---------|------|------|
| FixedThreadPool | LinkedBlockingQueue（无界） | 任务无限堆积 | OOM |
| CachedThreadPool | SynchronousQueue | 直接传递，无缓冲 | 线程数暴涨 |
| SingleThreadExecutor | LinkedBlockingQueue（无界） | 单线程+无限堆积 | OOM |

**生产环境建议**：
```java
// 推荐：有界队列 + 拒绝策略
new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),  // 有界，内存可控
    new ThreadPoolExecutor.CallerRunsPolicy() // 队列满时调用线程执行
);
```

- 始终使用**有界队列**防止OOM
- 配置**拒绝策略**处理队列满的情况
- 避免使用Executors的便捷方法（隐藏风险）

