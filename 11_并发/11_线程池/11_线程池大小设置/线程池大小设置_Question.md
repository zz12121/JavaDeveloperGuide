---
type: Question
chapter: 11_线程池
topic: 11_线程池大小设置
difficulty: 中等
tags: [线程池, 线程数, CPU密集型, IO密集型, 压测, 调优]
---

# 线程池大小设置 - 面试题

---

## Q1: CPU 密集型任务的线程数应该如何设置？

**答案**：

```
线程数 = CPU 核数 + 1（或 CPU 核数）
```

**原因**：
- CPU 密集型任务一直在计算，线程数过多导致频繁上下文切换
- 线程数 = CPU 核数时，每个核心一个线程，CPU 利用率最高
- +1 是为了防止某线程因缺页中断等原因暂停，让额外的线程顶上

**代码获取 CPU 核数**：
```java
int cpuCores = Runtime.getRuntime().availableProcessors();
int poolSize = cpuCores + 1;
```

---

## Q2: IO 密集型任务的线程数应该如何设置？

**答案**：

**简化公式**：
```
线程数 = CPU 核数 × 2（推荐范围 2~10 倍）
```

**精确公式**：
```
线程数 = CPU 核数 × (1 + IO等待时间 / CPU计算时间)

// 或考虑目标 CPU 利用率
N = CPU × U × (1 + W/C)
U = 目标 CPU 利用率（0~1）
W = 等待时间
C = 计算时间
```

**原因**：
- IO 密集型任务大部分时间在等待（网络、磁盘），不占用 CPU
- 增加线程数可以在等待期间让其他线程使用 CPU
- 但不能无限增大，受限于内存、连接数、上下文切换开销

**经验值**：IO 密集型线程数一般不超过 200~500

---

## Q3: 混合型任务（既有 CPU 又有 IO）如何处理？

**答案**：

**方案一：拆分为两个线程池**
```java
// CPU 密集型池
ExecutorService cpuPool = Executors.newFixedThreadPool(cpuCores + 1);

// IO 密集型池
ExecutorService ioPool = new ThreadPoolExecutor(
    cpuCores * 2,
    cpuCores * 10,
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000)
);
```

**方案二：使用统一公式**
```
线程数 = CPU 核数 × (1 + 平均等待时间 / 平均计算时间)
```

**方案三：CompletableFuture 组合**
```java
// CPU 密集型操作在 CPU 池
CompletableFuture.supplyAsync(() -> compute(), cpuPool)
    // IO 密集型操作在 IO 池
    .thenApplyAsync(result -> fetchData(result), ioPool)
    .thenApplyAsync(data -> process(data), cpuPool);
```

---

## Q4: 如何获取正确的 CPU 核数？在容器环境有什么注意事项？

**答案**：

**获取逻辑核数**：
```java
int cores = Runtime.getRuntime().availableProcessors();
```

**注意事项**：
1. **逻辑核 vs 物理核**：超线程下逻辑核 = 物理核 × 2
2. **Docker 容器问题**：JDK 10 之前会获取宿主机的核数，而非容器限制的核数
3. **JDK 10+ 解决方案**：
   ```bash
   # 使用 -XX:ActiveProcessorCount 覆盖
   java -XX:ActiveProcessorCount=4 -jar app.jar
   ```

**容器环境正确获取**：
```java
// JDK 10+ 使用 Container API
int containerCores = (int) ProcessHandle.current()
    .info().commandLine()
    .map(cmd -> /* 解析 cgroup 限制 */)
    .orElse(Runtime.getRuntime().availableProcessors());
```

---

## Q5: 线程池大小设置过大或过小分别有什么问题？

**答案**：

**线程数过小**：
- CPU 利用率低，资源浪费
- 任务排队时间增加，响应延迟升高
- 吞吐量达不到预期

**线程数过大**：
- 上下文切换开销剧增（每次切换约 1~10μs）
- 内存占用增加（每个线程栈默认 1MB）
- 数据库连接池、文件句柄等资源耗尽
- 可能触发 OOM

**平衡原则**：
- 队列长度与线程数需要平衡
- 队列太大 → 任务延迟高，maximumPoolSize 不生效
- 队列太小 → 频繁触发拒绝策略
- **没有银弹，需要压测调优**

---

## Q6: 如何进行线程池压测调优？

**答案**：

**压测步骤**：

1. **确定目标**：QPS、RT、成功率
2. **初始配置**：基于公式估算初始值
3. **逐步调整**：根据压测结果调整参数
4. **验证边界**：测试极限场景

**调优示例**：
```
场景：HTTP 接口调用下游服务，平均耗时 100ms，目标 QPS 1000

初始：core=10, max=20, queue=100
结果：QPS=80, CPU=15%
→ 线程数太少，增加线程数

调整：core=50, max=100, queue=500
结果：QPS=450, CPU=60%
→ 有改善，继续增加

调整：core=100, max=200, queue=1000
结果：QPS=950, CPU=85%
→ 接近目标，微调

最终：core=120, max=200, queue=2000
结果：QPS=1050, CPU=90%
✅ 满足目标
```

---

## Q7: 线程池参数可以动态调整吗？如何调整？

**答案**：

**可以动态调整的参数**：
```java
ThreadPoolExecutor executor = ...;

// 调整核心线程数
executor.setCorePoolSize(100);

// 调整最大线程数
executor.setMaximumPoolSize(200);

// 调整存活时间
executor.setKeepAliveTime(60, TimeUnit.SECONDS);

// 调整拒绝策略
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

**无法动态调整的参数**：
- 队列类型和容量（需要重新创建线程池）
- ThreadFactory（需要重新创建线程池）

**动态调整场景**：
- 大促期间临时扩容
- 根据负载自动扩缩容
- 配置中心热更新

---

## Q8: 生产环境线程池大小设置的经验法则是什么？

**答案**：

| 场景 | 推荐配置 | 说明 |
|------|---------|------|
| 纯 CPU 计算 | core = N+1, max = N+1 | N = CPU 核数 |
| 轻量 IO | core = N×2, max = N×4 | 如缓存读取 |
| 中等 IO | core = N×4, max = N×8 | 如数据库查询 |
| 重度 IO | core = N×8, max = N×16 | 如外部 HTTP 调用 |
| 混合场景 | 拆分为两个池 | CPU 池 + IO 池 |

**队列大小建议**：
- 有界队列：100~10000（根据任务量）
- 队列太小 → 频繁触发拒绝策略
- 队列太大 → 内存占用高，响应延迟

**关键原则**：
1. 先估算，再压测，后上线
2. 保留监控和动态调整能力
3. 根据实际负载持续优化

---
