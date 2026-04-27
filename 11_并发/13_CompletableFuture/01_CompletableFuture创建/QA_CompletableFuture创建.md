
# CompletableFuture创建

## Q1：CompletableFuture 有哪些创建方式？

**A**：
1. **supplyAsync(Supplier)** — 有返回值的异步任务，返回 `CompletableFuture<T>`
2. **runAsync(Runnable)** — 无返回值的异步任务，返回 `CompletableFuture<Void>`
3. **completedFuture(T)** — 创建已完成的 CF（测试用）
4. **new CompletableFuture<>()** — 创建未完成的 CF，手动调用 `complete()` 完成

前两个可以指定 Executor，不指定则默认使用 ForkJoinPool.commonPool。

---

## Q2：supplyAsync 和 runAsync 的区别是什么？

**A**：
- `supplyAsync` 接收 `Supplier<T>`，有返回值，适用于**需要结果**的场景（如查询数据库、调用接口）
- `runAsync` 接收 `Runnable`，无返回值，适用于**只执行动作**的场景（如发送通知、写入日志）

选择标准：异步任务是否需要返回结果。

---

## Q3：CompletableFuture 默认使用什么线程池？有什么注意事项？

**A**：默认使用 **ForkJoinPool.commonPool**（并行度 = CPU 核心数 - 1）。

注意事项：
- CPU 密集型短任务可以用默认池
- I/O 阻塞任务（HTTP 请求、数据库查询）**必须用自定义线程池**，否则会阻塞 commonPool 影响其他并行操作（如 parallel stream）
- 自定义池推荐用 ThreadPoolExecutor，根据 I/O 等待时间设置合理线程数

---
```java
// 方式1：supplyAsync（有返回值）
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
    return fetchData();
});

// 方式2：runAsync（无返回值）
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
    log.info("异步任务");
});

// 方式3：指定线程池
ExecutorService pool = Executors.newFixedThreadPool(4);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> {
    return fetchData();
}, pool);

// 方式4：手动完成
CompletableFuture<String> cf4 = new CompletableFuture<>();
cf4.complete("手动结果");           // 正常完成
// cf4.completeExceptionally(new RuntimeException("失败"));  // 异常完成
```

