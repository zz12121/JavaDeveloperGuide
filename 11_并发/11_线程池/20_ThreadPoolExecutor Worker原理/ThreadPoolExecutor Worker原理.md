
# ThreadPoolExecutor Worker原理

## 核心结论

Worker 是 ThreadPoolExecutor 的内部类，**继承 AQS 实现独占锁 + 实现 Runnable 接口**。每个 Worker 就是一个工作线程，通过 `runWorker()` 方法循环从队列中获取任务（`getTask()`）并执行。Worker 继承 AQS 的目的是精确控制线程中断——只有在获取锁（正在执行任务）时才允许被中断。

## 深度解析

### Worker 类设计

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;          // Worker 对应的工作线程
    Runnable firstTask;           // 第一个任务（可为 null）
    volatile long completedTasks; // 已完成任务计数

    Worker(Runnable firstTask) {
        setState(-1);   // 初始 state=-1，禁止在 runWorker 前被中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }

    // AQS 实现：独占锁
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {  // state: 0=空闲, 1=执行中
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    // 判断是否锁定（正在执行任务）
    public boolean isLocked() {
        return isHeldExclusively();
    }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try { t.interrupt(); } catch (SecurityException ignore) {}
        }
    }
}
```

### 为什么 Worker 继承 AQS

Worker 继承 AQS 不是为了锁竞争，而是为了**精确控制中断时机**：

| state | 含义 | 是否可中断 |
|-------|------|-----------|
| -1 | 初始状态，runWorker 尚未启动 | ❌ 不可中断 |
| 0 | 空闲（等待获取任务） | ✅ 可中断 |
| 1 | 正在执行任务 | ❌ 不可中断 |

- **空闲时可以中断**：线程阻塞在 getTask() 的 poll/take，可以响应 shutdownNow
- **执行任务时不可中断**：保证任务执行完整性

### addWorker() 流程

```
addWorker(firstTask, core):
  1. CAS 更新线程池状态（RUNNING 或 SHUTDOWN）
  2. CAS 增加 workerCount（ctl 高3位=状态，低29位=线程数）
  3. 创建 Worker 对象（含 firstTask + Thread）
  4. 加锁（workers 集合）
  5. workers.add(worker)
  6. 启动 worker.thread.start()
  7. 解锁
```

### runWorker() 核心逻辑

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();           // state: -1 → 0，允许被中断
    boolean completedAbruptly = true;
    try {
        // 循环获取任务：先执行 firstTask，再从队列取
        while (task != null || (task = getTask()) != null) {
            w.lock();      // state: 0 → 1，执行任务中不可中断
            try {
                beforeExecute(wt, task);  // 钩子方法
                try {
                    task.run();            // 执行任务
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();  // state: 1 → 0，任务执行完毕
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);  // 线程退出清理
    }
}
```

### getTask() —— 线程回收的关键

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        // 1. 检查线程池状态，非RUNNING/SHUTDOWN 返回 null
        // 2. 判断是否需要超时控制
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        
        if (timed) {
            // 非核心线程：poll(keepAliveTime) 超时返回 null → 线程退出
            r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
        } else {
            // 核心线程：take() 阻塞等待
            r = workQueue.take();
        }
        
        if (r != null) return r;
        timedOut = true;  // 超时
    }
}
```

### 线程回收流程

```
getTask() 返回 null
  → runWorker 退出 while 循环
  → processWorkerExit(w, false)
    → CAS 减少 workerCount
    → 从 workers 集合移除
    → 尝试 terminate（如果线程池为 SHUTDOWN 且无任务）
```

## 易错点/踩坑

- ❌ 以为 Worker 继承 AQS 是为了锁竞争
- ✅ Worker 继承 AQS 是为了精确控制线程中断时机
- ❌ 以为核心线程永远不会被回收
- ✅ 设置 `allowCoreThreadTimeOut(true)` 后核心线程也会超时回收
- ❌ 以为线程池 shutdown 后任务还能提交
- ✅ SHUTDOWN 状态不接受新任务，但会执行队列中已有任务

## 代码示例

```java
// 打印线程池内部 Worker 信息
ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(4);

// 通过反射获取 workers 集合（仅调试用）
Field workersField = ThreadPoolExecutor.class.getDeclaredField("workers");
workersField.setAccessible(true);
Set<Worker> workers = (Set<Worker>) workersField.get(executor);

log.info("活跃线程数={}, 池大小={}, 队列大小={}",
    executor.getActiveCount(),
    executor.getPoolSize(),
    executor.getQueue().size());
```

## 图解/流程

```
Worker 生命周期：
  创建 Worker → 启动线程 → runWorker()
                              │
                              ├── task = firstTask → lock() → task.run() → unlock()
                              │
                              ├── task = getTask()  → lock() → task.run() → unlock()
                              │       │                                        │
                              │       ├── poll(keepAlive) → null → 线程退出    │
                              │       └── take() → 阻塞等待新任务 ────────────→│
                              │
                              └── getTask() == null → processWorkerExit() → 线程回收
```

## 关联知识点
