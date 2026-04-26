
# newFixedThreadPool

## Q1：newFixedThreadPool 的参数是什么？

**A**：

```java
new ThreadPoolExecutor(
    nThreads, nThreads,        // core = max
    0L, TimeUnit.MILLISECONDS, // 无超时
    new LinkedBlockingQueue<>()  // 无界队列
);
```

核心线程数 = 最大线程数 = nThreads，无界队列。

---

## Q2：newFixedThreadPool 有什么问题？

**A**：使用**无界 LinkedBlockingQueue**，任务堆积时 OOM。

因为 core = max，队列满时不会创建新线程。所有多余任务都堆积在队列中。

**解决方案**：手动创建 ThreadPoolExecutor，使用有界队列：
```java
new ThreadPoolExecutor(10, 10, 0L, MILLISECONDS, new ArrayBlockingQueue<>(100));
```

