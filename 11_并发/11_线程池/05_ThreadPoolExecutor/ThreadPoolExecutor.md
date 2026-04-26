
# ThreadPoolExecutor

## 核心结论

ThreadPoolExecutor 是 JUC 线程池的核心实现类。继承 AbstractExecutorService，实现了 execute/submit/shutdown 等方法。**阿里巴巴规范要求手动创建 ThreadPoolExecutor**，禁止使用 Executors 工厂方法。

## 深度解析

### 继承关系

```
Executor（execute）
  └── ExecutorService（submit/shutdown/invokeAll）
       └── AbstractExecutorService（submit 默认实现）
            └── ThreadPoolExecutor（核心实现）
```

### Worker 内部类

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;
    
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
    
    public void run() { runWorker(this); }
}
```

- Worker 继承 AQS，实现**不可重入的独占锁**
- state=0 未锁定，state=1 已锁定（执行任务时）
- 不可重入：调用 shutdown 时可以中断正在等待任务的线程

### runWorker

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();       // 执行任务时加锁
            try {
                beforeExecute(wt, task); // 钩子方法
                try {
                    task.run();           // 执行任务
                    afterExecute(task, null); // 钩子方法
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
    } finally {
        processWorkerExit(w, false);
    }
}
```

### 钩子方法

```java
// 可在子类中覆盖
protected void beforeExecute(Thread t, Runnable r) { }
protected void afterExecute(Runnable r, Throwable t) { }
protected void terminated() { }
```

### 推荐创建方式

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // corePoolSize
    20,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,           // keepAliveTime
    new LinkedBlockingQueue<>(1000),  // 有界队列
    new ThreadFactoryBuilder().setNameFormat("biz-pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

## 易错点与踩坑

### 1. 禁止使用 Executors 工厂方法（阿里巴巴规范）
```java
// ❌ 禁止
ExecutorService pool = Executors.newFixedThreadPool(10);
ExecutorService pool = Executors.newCachedThreadPool();
ExecutorService pool = Executors.newSingleThreadExecutor();

// ✅ 推荐：手动创建 ThreadPoolExecutor
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    10, 20, 60, SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadFactoryBuilder().setNameFormat("biz-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 2. Worker 继承 AQS 的不可重入设计
- Worker 用 AQS 实现独占锁，state=0 未锁定，state=1 锁定
- **不可重入**：任务执行中调用 `execute()` 不会死锁，因为 `shutdown()` 能中断等待中的线程
- 如果是可重入锁，`shutdown()` 无法中断正在执行任务的线程

### 3. beforeExecute/afterExecute 的异常处理
```java
// beforeExecute 抛异常 → 任务不会执行，线程死亡退出
// afterExecute 抛异常 → 任务已执行完，但线程死亡退出
// 所以 afterExecute 中不要抛异常，要 try-catch 处理
@Override
protected void afterExecute(Runnable r, Throwable t) {
    try {
        // 处理逻辑
    } catch (Exception e) {
        log.error("afterExecute error", e); // 不要让异常逃逸
    }
}
```

### 4. 线程池关闭后不能再提交任务
```java
pool.shutdown();
pool.execute(() -> {}); // 抛 RejectedExecutionException
// 状态非 RUNNING 时，所有新任务都会被拒绝
```
## 关联知识点

