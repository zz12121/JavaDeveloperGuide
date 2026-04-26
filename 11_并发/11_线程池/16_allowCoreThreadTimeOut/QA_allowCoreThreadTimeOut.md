
# allowCoreThreadTimeOut

## Q1：allowCoreThreadTimeOut 做了什么？

**A**：允许核心线程在空闲超过 keepAliveTime 后被回收。

默认情况下，核心线程用 `take()` 阻塞等待，永远不会超时退出。开启后，核心线程改用 `poll(keepAliveTime)`，超时后返回 null，线程退出。

线程池空闲后可以缩容到 0。

---

## Q2：CachedThreadPool 和 allowCoreThreadTimeOut 有什么关系？

**A**：CachedThreadPool 的 corePoolSize=0，本质上就是所有线程都允许超时回收。

`allowCoreThreadTimeOut(true)` 是在非零 corePoolSize 的线程池上实现类似效果。

---

## Q3：什么时候用 allowCoreThreadTimeOut？

**A**：任务量波动大、空闲期长的场景：

- 白天任务多、夜间空闲的后台服务
- 弹性伸缩需求
- 需要节省 CPU 和内存资源

代价是首次任务有冷启动延迟（需要重新创建线程）。

---
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(4, 8, 30, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100));

// 默认：核心线程永不回收（用 take() 阻塞等待）
pool.allowCoreThreadTimeOut(true);
// 开启后：核心线程改用 poll(30s)，空闲 30 秒后退出

// 适用场景：白天繁忙、夜间空闲的服务
// 代价：首次任务有冷启动延迟（需重新创建线程）
```


