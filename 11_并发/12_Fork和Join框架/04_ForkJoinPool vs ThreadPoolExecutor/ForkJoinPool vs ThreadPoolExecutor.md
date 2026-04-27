
# ForkJoinPool vs ThreadPoolExecutor

## 核心结论

ForkJoinPool 和 ThreadPoolExecutor 都是线程池，但设计理念不同：ThreadPoolExecutor 处理**独立任务**（共享队列），ForkJoinPool 处理**可拆分的父子任务**（每线程独立队列 + 工作窃取）。选择依据：任务是否可以递归拆分。

## 深度解析

### 全面对比

| 维度 | ThreadPoolExecutor | ForkJoinPool |
|------|-------------------|--------------|
| **任务模型** | 独立任务，无父子关系 | 父子任务，递归拆分 |
| **任务队列** | 全局共享阻塞队列 | 每线程独立双端队列 |
| **调度策略** | 任务从共享队列竞争获取 | 工作窃取（头取尾偷） |
| **适用场景** | Web 请求、定时任务、通用并发 | 分治算法、递归计算、并行流 |
| **负载均衡** | 依赖队列竞争 | 工作窃取自动均衡 |
| **典型用例** | HTTP 线程池、数据库连接池 | 数组排序、MapReduce、Stream |
| **线程利用率** | 任务不均时可能空闲 | 窃取机制减少空闲 |
| **任务粒度** | 粗粒度即可 | 需合理拆分（阈值控制） |

### 架构对比

```
ThreadPoolExecutor:
┌─────────────────────────┐
│    全局 BlockingQueue     │
│  [T1][T2][T3][T4][T5]  │  ← 所有线程竞争同一个队列
└──┬──┬──┬──┬─────────────┘
   │  │  │  │
  W1 W2 W3 W4  ← Worker Thread

ForkJoinPool:
┌──────────┐ ┌──────────┐ ┌──────────┐
│ W1 Queue │ │ W2 Queue │ │ W3 Queue │
│ [T1][T2] │ │ [T3][T4] │ │   空！   │  ← 每线程独立队列
│ ↑取  ↓偷─│─│ ↑取      │ │ ↑偷─────│
└──────────┘ └──────────┘ └──────────┘
```

### 如何选择

**用 ThreadPoolExecutor 当**：
- 任务之间相互独立
- I/O 密集型任务（如数据库查询、HTTP 调用）
- 需要阻塞队列的各种特性（容量限制、优先级等）
- 服务端请求处理（Tomcat、Dubbo 等框架底层都用 TPE）

**用 ForkJoinPool 当**：
- 大任务可拆分为相似的子任务（分治思想）
- CPU 密集型的计算任务
- 需要递归并行处理（归并排序、快速排序）
- 使用并行流（parallel stream 默认用 commonPool）

### 混合使用的注意

```java
// ❌ 在 ForkJoinPool 内部使用阻塞操作
// 线程被阻塞后无法处理窃取，其他线程的任务也受影响
class BadTask extends RecursiveTask<Void> {
    protected Void compute() {
        Thread.sleep(1000);  // 阻塞，浪费工作线程
        return null;
    }
}

// ✅ I/O 密集任务用 ThreadPoolExecutor
ExecutorService ioPool = Executors.newFixedThreadPool(10);
```

## 关联知识点

