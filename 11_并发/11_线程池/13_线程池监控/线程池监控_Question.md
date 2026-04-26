---
type: Question
chapter: 11_线程池
topic: 13_线程池监控
difficulty: 中等
tags: [线程池, JMX, Prometheus, 监控, 生产环境]
---

# 线程池监控 - 面试题

---

## Q1: ThreadPoolExecutor 提供了哪些核心监控方法？

**答案**：

| 方法 | 说明 | 典型用途 |
|------|------|---------|
| `getActiveCount()` | 正在执行任务的线程数 | 当前负载 |
| `getPoolSize()` | 当前线程池线程数 | 线程创建情况 |
| `getQueue().size()` | 等待队列中的任务数 | 积压检测 |
| `getTaskCount()` | 总任务数 | 累计任务量 |
| `getCompletedTaskCount()` | 已完成任务数 | 吞吐量计算 |
| `getLargestPoolSize()` | 历史最大线程数 | 峰值分析 |
| `isShutdown()` | 是否已关闭 | 关闭状态 |
| `isTerminated()` | 是否已终止 | 终止状态 |

**注意**：`getTaskCount() - getCompletedTaskCount()` 不等于"待处理任务数"，还需减去 `getActiveCount()`。

---

## Q2: 如何通过 JMX 监控线程池？

**答案**：

```java
// 注册线程池到 JMX
ThreadPoolExecutor executor = new ThreadPoolExecutor(...);
MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
ObjectName objectName = new ObjectName("com.example:type=ThreadPool,name=BusinessPool");
mBeanServer.registerMBean(new StandardMBean(executor, ThreadPoolExecutorMXBean.class), objectName);
```

**查看方式**：
- JConsole：JDK 自带，连接进程后查看 MBean 标签
- VisualVM：插件丰富，可监控多个指标
- jmxterm：命令行工具

**可查看指标**：ActiveCount、PoolSize、QueueSize、CompletedTaskCount、CorePoolSize、MaximumPoolSize 等。

---

## Q3: Spring Boot 如何集成 Prometheus 监控线程池？

**答案**：

**1. 添加依赖**：
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**2. 注册指标**：
```java
@Component
public class ThreadPoolMetrics {
    @Autowired
    private MeterRegistry meterRegistry;
    
    @PostConstruct
    public void init() {
        ThreadPoolExecutor executor = ...;
        
        Gauge.builder("threadpool.active", executor, ThreadPoolExecutor::getActiveCount)
            .tag("name", "business")
            .register(meterRegistry);
            
        Gauge.builder("threadpool.queue.size", executor, e -> e.getQueue().size())
            .tag("name", "business")
            .register(meterRegistry);
    }
}
```

**3. 访问端点**：`/actuator/prometheus`

---

## Q4: 线程池监控的常见告警规则有哪些？

**答案**：

**Prometheus 告警规则示例**：

```yaml
groups:
  - name: threadpool_alerts
    rules:
      # 队列积压告警
      - alert: ThreadPoolQueueFull
        expr: threadpool_queue_size / threadpool_queue_capacity > 0.8
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "线程池队列积压超过80%"
          
      # 线程池满载告警
      - alert: ThreadPoolActiveHigh
        expr: threadpool_active / threadpool_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "线程池活跃线程接近上限"
          
      # 任务拒绝告警
      - alert: ThreadPoolRejection
        expr: increase(threadpool_rejected_total[1m]) > 0
        labels:
          severity: critical
```

---

## Q5: 如何通过钩子方法实现任务级监控？

**答案**：

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(...) {
    private final ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        startTime.set(System.currentTimeMillis());
    }
    
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        long duration = System.currentTimeMillis() - startTime.get();
        startTime.remove();
        
        // 记录任务执行时间
        metrics.recordTaskDuration(duration);
        
        if (t != null) {
            metrics.incrementErrorCount();
            log.error("任务执行异常", t);
        }
    }
    
    @Override
    protected void terminated() {
        super.terminated();
        log.info("线程池已终止");
    }
};
```

**注意**：`afterExecute` 中的 `r` 可能被包装为 `FutureTask`，无法直接获取原始任务信息。

---

## Q6: 如何计算线程池的实际吞吐量？

**答案**：

```java
@Component
public class ThreadPoolThroughputMetrics {
    
    private long lastCompleted = 0;
    private long lastTimestamp = System.currentTimeMillis();
    
    @Scheduled(fixedRate = 60000) // 每分钟计算一次
    public void calculateThroughput() {
        long currentCompleted = executor.getCompletedTaskCount();
        long currentTimestamp = System.currentTimeMillis();
        
        long completedDelta = currentCompleted - lastCompleted;
        long timeDelta = (currentTimestamp - lastTimestamp) / 1000;
        
        double throughput = (double) completedDelta / timeDelta; // 每秒处理任务数
        
        lastCompleted = currentCompleted;
        lastTimestamp = currentTimestamp;
        
        log.info("线程池吞吐量: {} tasks/sec", throughput);
    }
}
```

---

## Q7: 监控线程池时有哪些常见陷阱？

**答案**：

| 陷阱 | 说明 | 解决方案 |
|------|------|---------|
| 监控数据延迟 | 读取的是某一时刻快照，非实时精确值 | 多次采样取平均 |
| 监控线程泄漏 | 用 Executors 创建监控线程池不关闭 | 复用 Spring @Scheduled 或 JMX |
| afterExecute 信息丢失 | submit() 的任务被包装为 FutureTask | 用 ThreadLocal/MDC 传递上下文 |
| 误读 TaskCount | 不是"待处理任务数" | 计算公式：taskCount - completed - active |
| 队列容量误判 | LinkedBlockingQueue 无界时 capacity 为 Integer.MAX_VALUE | 使用有界队列并设置合理容量 |

---

## Q8: 生产环境线程池监控的推荐采集频率是什么？

**答案**：

| 指标 | 采集频率 | 用途 |
|------|---------|------|
| active/poolSize | 10s | 实时负载监控 |
| queueSize | 10s | 积压检测 |
| completedTaskCount | 60s | 吞吐量计算 |
| taskCount | 60s | 任务总量统计 |
| largestPoolSize | 300s | 峰值分析 |

**原因**：
- 活跃线程和队列大小变化快，需要高频采集
- 累计类指标（completed/taskCount）低频采集即可，计算差值得到速率
- 峰值类指标（largestPoolSize）变化慢，低频采集减少开销

---
