
# newCachedThreadPool

## 核心结论

缓存线程池：核心线程 0，最大线程 Integer.MAX_VALUE，60 秒超时回收。使用 SynchronousQueue，任务来就创建线程。**高并发时线程数爆炸**。

## 深度解析

### 创建参数

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(
        0,                          // 核心线程 0
        Integer.MAX_VALUE,           // ⚠️ 最大线程无上限
        60L, TimeUnit.SECONDS,      // 60秒超时
        new SynchronousQueue<Runnable>() // 零容量队列
    );
}
```

### 行为特征

```
提交任务：
  无空闲线程 → 创建新线程（不限数量）
  有空闲线程 → 复用
  线程空闲60秒 → 自动回收
  
高并发：线程数 = 任务数（可能几万个线程）
```

### 适用场景

- 短生命周期任务（大量小任务）
- 突发流量处理
- 异步任务，不需要控制并发度

### 风险

- **线程数爆炸**：任务提交速度快于处理速度时，不断创建线程
- **OOM**：每个线程约 1MB 栈空间，万级线程就吃掉 GB 内存
- **CPU 上下文切换**：线程过多导致调度开销大

## 易错点与踩坑

### 1. 线程数爆炸（核心考点）
```java
// ❌ 高并发场景下，线程数 = 任务并发数
// 如果 10000 个任务同时提交 → 创建 10000 个线程
Executors.newCachedThreadPool();

// 每个线程栈默认 1MB → 10000 线程 = 10GB 内存
// 频繁上下文切换 → CPU 利用率反而下降
```

### 2. SynchronousQueue 的特性容易被忽略
- SynchronousQueue 没有容量，`offer()` 只有在有消费者 `poll()` 时才返回 true
- 意味着每个任务必须有一个空闲线程来接收，否则就创建新线程
- **没有"缓冲"能力**，任务提交速度 > 处理速度 → 疯狂创建线程

### 3. corePoolSize = 0 的冷启动问题
- 空闲 60 秒后所有线程被回收，线程数降到 0
- 新任务来时需要重新创建线程，有冷启动延迟
- 如果对延迟敏感，可以考虑 `prestartCoreThread()`（但 core=0 时无效）

### 4. 适合与不适合的场景
```
✅ 适合：短生命周期任务、突发流量、任务执行快
❌ 不适合：长时间运行的任务、高并发持续流量、对线程数敏感
```
## 关联知识点

