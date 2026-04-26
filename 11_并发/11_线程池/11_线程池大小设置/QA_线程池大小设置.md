
# 线程池大小设置

## Q1：线程池大小怎么设置？

**A**：

- **CPU 密集型**：`CPU 核数 + 1`
- **IO 密集型**：`CPU 核数 × 2`（推荐范围 2~10 倍）
- **混合型**：`CPU 核数 × (1 + 平均等待时间 / 平均计算时间)`

核心思想：CPU 密集型避免上下文切换，IO 密集型利用等待时间。

---

## Q2：如何获取 CPU 核数？

**A**：`Runtime.getRuntime().availableProcessors()`

返回逻辑核心数（包含超线程），一般作为线程池大小的基准。

---

## Q3：IO 密集型线程池可以设置很大吗？

**A**：**理论上可以，但不建议无限制**。

线程数过多的问题：
- 内存消耗：每个线程约 1MB 栈空间
- 上下文切换开销：调度成本随线程数增加
- 连接数限制：数据库连接、文件句柄等有限

建议配合有界队列控制总任务量：`new ThreadPoolExecutor(cores, cores*2, 60s, new ArrayBlockingQueue<>(1000))`

---
```java
int cpuCores = Runtime.getRuntime().availableProcessors();
System.out.println("CPU 核数: " + cpuCores);

// CPU 密集型
int cpuBound = cpuCores + 1;

// IO 密集型
int ioBound = cpuCores * 2;

// 混合型（估算公式）
int mixed = (int) (cpuCores * (1 + avgWaitTime / avgComputeTime));

ThreadPoolExecutor pool = new ThreadPoolExecutor(
    cpuCores, cpuCores * 2, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000));
```

