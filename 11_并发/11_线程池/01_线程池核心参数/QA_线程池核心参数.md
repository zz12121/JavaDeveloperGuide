
# 线程池核心参数

## Q1：ThreadPoolExecutor 的 7 个核心参数是什么？

**A**：

1. **corePoolSize**：核心线程数，常驻不回收
2. **maximumPoolSize**：最大线程数（核心 + 非核心）
3. **keepAliveTime**：非核心线程空闲存活时间
4. **unit**：时间单位
5. **workQueue**：任务等待队列
6. **threadFactory**：线程创建工厂
7. **handler**：拒绝策略（队列满 + 线程满时触发）

---

## Q2：corePoolSize 和 maximumPoolSize 有什么关系？

**A**：

- `corePoolSize ≤ maximumPoolSize`
- 核心线程数 = 常驻线程（即使空闲也不回收）
- 非核心线程数 = maximumPoolSize - corePoolSize
- 非核心线程空闲超过 keepAliveTime 后被回收

---

## Q3：为什么要自定义 ThreadFactory？

**A**：

1. **命名线程**：方便排查问题（默认 "pool-N-thread-M"）
2. **设置守护线程**：控制 JVM 退出行为
3. **设置优先级**：不同线程池不同优先级
4. **绑定上下文**：设置 UncaughtExceptionHandler

---
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize
    8,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,          // keepAliveTime
    new ArrayBlockingQueue<>(1000),  // workQueue
    new ThreadFactory() {           // threadFactory
        private int count = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "pool-" + count++);
            t.setUncaughtExceptionHandler((th, ex) -> log.error("线程异常", ex));
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // handler
);
```

