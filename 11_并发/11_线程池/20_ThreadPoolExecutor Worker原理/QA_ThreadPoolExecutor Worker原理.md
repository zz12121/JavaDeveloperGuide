
# ThreadPoolExecutor Worker原理

## Q1：ThreadPoolExecutor 的 Worker 是怎么设计的？为什么继承 AQS？

**A**：Worker 是 ThreadPoolExecutor 的内部类，有两个关键设计：

1. **继承 AQS**：不是为了锁竞争，而是精确控制线程中断时机
   - `state = -1`：初始状态，禁止中断
   - `state = 0`：空闲，可以中断（等待任务时）
   - `state = 1`：执行任务中，不可中断
2. **实现 Runnable**：Worker 本身就是线程的任务，`run()` 方法内部调用 `runWorker()`

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    
    Worker(Runnable firstTask) {
        setState(-1);  // 禁止在runWorker前被中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
    
    public void run() { runWorker(this); }
}
```

---

## Q2：runWorker 的执行流程是怎样的？

**A**：

```
runWorker(w):
  1. w.unlock() → state: -1 → 0，允许被中断
  2. while (task != null || (task = getTask()) != null):
       a. w.lock() → 获取锁（标记正在执行任务）
       b. beforeExecute() → 前置钩子
       c. task.run() → 执行任务
       d. afterExecute() → 后置钩子
       e. w.unlock() → 释放锁
  3. processWorkerExit() → 清理退出
```

核心逻辑：**先执行 firstTask，然后循环调用 getTask() 从队列获取任务**。getTask() 返回 null 时线程退出。

---

## Q3：线程池中的线程是怎么回收的？

**A**：通过 `getTask()` 方法控制：

1. **核心线程**：调用 `workQueue.take()`，阻塞等待，不会回收
2. **非核心线程**：调用 `workQueue.poll(keepAliveTime, TimeUnit)`，超时返回 null
3. `getTask()` 返回 null → `runWorker()` 退出循环 → `processWorkerExit()` 回收

```
非核心线程回收：
  getTask() → poll(keepAliveTime) 超时 → 返回 null
  → runWorker 退出 → processWorkerExit
    → CAS 减少 workerCount → 从 workers 移除 Worker
```

特殊：调用 `allowCoreThreadTimeOut(true)` 后，核心线程也会超时回收。

