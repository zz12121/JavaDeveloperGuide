
# newFixedThreadPool

## 核心结论

固定大小线程池，核心线程数 = 最大线程数。使用**无界 LinkedBlockingQueue**，任务堆积时可能 OOM。

## 深度解析

### 创建参数

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(
        nThreads, nThreads,          // core = max = nThreads
        0L, TimeUnit.MILLISECONDS,   // 非核心不超时（因为 core=max）
        new LinkedBlockingQueue<Runnable>() // ⚠️ 无界队列（Integer.MAX_VALUE）
    );
}
```

### 行为特征

- 核心线程数 = 最大线程数 = nThreads
- keepAliveTime = 0（无意义，因为没有非核心线程）
- 无界队列 → maximumPoolSize 形同虚设（队列永不满，不会创建额外线程）
- 任务堆积在队列中，可能导致 OOM

### 线程数变化

```
提交任务：
  线程数 < n → 创建核心线程
  线程数 = n → 入队（永远不创建超过 n 个线程）
  队列增长 → OOM 风险
```

## 易错点与踩坑

### 1. 无界队列 OOM（核心考点）
```java
// ❌ 无界队列，任务堆积导致 OOM
Executors.newFixedThreadPool(10);
// 等价于 new LinkedBlockingQueue()，容量 Integer.MAX_VALUE
// 高峰期任务大量堆积 → 内存耗尽

// ✅ 使用有界队列
new ThreadPoolExecutor(
    10, 10,
    0L, MILLISECONDS,
    new LinkedBlockingQueue<>(1000), // 有界队列
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 2. maximumPoolSize 永远不生效
- 因为 core = max = nThreads，且使用无界队列
- 线程数始终 = nThreads，永远不创建额外线程
- 拒绝策略永远不触发（队列不满 → 不创建非核心 → 不拒绝）

### 3. 异常任务不影响后续任务
- 每个线程独立执行，一个任务异常只影响该线程的当前任务
- 线程不会死亡，会继续从队列取下一个任务执行
- **但如果用 submit → 不调用 get()，异常会被静默吞掉**

### 4. keepAliveTime = 0 的误导
- 不代表线程立即回收，而是因为 core=max，没有"非核心线程"
- 所有线程都是核心线程，默认不超时回收
## 关联知识点
