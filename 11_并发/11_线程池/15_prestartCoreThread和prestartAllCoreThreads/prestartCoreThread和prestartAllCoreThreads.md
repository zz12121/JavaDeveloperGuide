
# prestartCoreThread和prestartAllCoreThreads

## 核心结论

线程池默认是**懒加载**的：核心线程在首次提交任务时才创建。`prestartCoreThread()` 和 `prestartAllCoreThreads()` 可以提前启动核心线程。

## 深度解析

### 两个方法

```java
// 启动一个核心线程
public boolean prestartCoreThread() {
    return addWorker(null, true); // firstTask=null，从队列取任务
}

// 启动所有核心线程
public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true))
        ++n;
    return n;
}
```

### 为什么需要预热

```
不预热：
  首次提交任务 → 创建线程（耗时）→ 执行任务
  第一个任务响应延迟高

预热后：
  首次提交任务 → 线程已就绪 → 直接执行
  首个任务响应零延迟
```

### 适用场景

- 低延迟要求的系统（如交易系统）
- 启动时需要立即处理突发请求
- 避免首次请求的"冷启动"延迟

## 易错点与踩坑

### 1. prestartCoreThread 可能返回 false
```java
// 以下情况返回 false：
// 1. 线程数已经 >= corePoolSize
// 2. 线程池已 shutdown
// 3. ThreadFactory 创建线程失败（如 OOM）
boolean started = pool.prestartCoreThread();
if (!started) {
    log.warn("预热失败，当前线程数可能已满");
}
```

### 2. corePoolSize = 0 时 prestartCoreThread 无效
```java
// CachedThreadPool: corePoolSize = 0
// prestartCoreThread() → addWorker(null, true)
// addWorker 中判断 workerCountOf(ctl.get()) < corePoolSize
// 0 < 0 = false → 直接返回 false
// 所以 CachedThreadPool 无法预热
```

### 3. 预热后线程在等什么
```java
// 预热创建的线程执行 addWorker(null, true)
// firstTask = null，线程启动后直接调用 getTask()
// getTask() → queue.take() → 阻塞等待任务
// 所以预热后的线程处于 WAITING 状态，等待队列中的任务
```

### 4. 预热与 allowCoreThreadTimeOut 的冲突
```java
pool.prestartAllCoreThreads(); // 启动所有核心线程
pool.allowCoreThreadTimeOut(true); // 允许核心线程超时
// 如果预热后一直没有任务 → keepAliveTime 后线程被回收
// 预热白费了 → 两个配置不要同时使用
```
## 关联知识点
