
# thenApplyAsync

## Q1：thenApply 和 thenApplyAsync 有什么区别？

**A**：核心区别在于**执行线程**不同：

- `thenApply`：在上游任务完成的**同一线程**中执行转换，不涉及线程切换
- `thenApplyAsync`：将转换任务**提交到线程池**，在新的线程中执行

```java
// thenApply：如果 supplyAsync 用了 pool1，thenApply 也在 pool1 的线程中
// thenApplyAsync：提交到 commonPool（无参）或指定 Executor（有参）
```

**选择原则**：
- 转换操作很轻量（如简单计算）→ 用 `thenApply`，减少线程切换开销
- 转换操作耗时（如 IO、复杂计算）→ 用 `thenApplyAsync`，避免阻塞上游线程

---

## Q2：thenApplyAsync 不指定 Executor 时用什么线程池？

**A**：默认使用 `ForkJoinPool.commonPool()`。

这与 `supplyAsync`/`runAsync` 的无参版本行为一致。JDK8 中 commonPool 默认线程数为 `CPU 核心数 - 1`。

**注意**：如果 commonPool 已被并行流占满，所有 thenApplyAsync 任务都会排队等待。生产环境建议指定自定义 Executor。

---

## Q3：异步方法链中如何控制线程池切换？

**A**：通过不同位置使用 sync/async 方法精确控制：

```java
supplyAsync(() -> fetchData(), ioPool)       // IO 线程池
    .thenApplyAsync(data -> parse(data), cpuPool)  // CPU 线程池
    .thenAccept(result -> save(result));      // 回到上游（cpuPool）线程
```

- 需要切换线程池 → 用 `xxxAsync` + 指定 Executor
- 继续在当前线程执行 → 用 `xxx`（同步版本）

---

## Q4：所有 Callback 方法都有 Async 版本吗？

**A**：几乎都有，只有 `exceptionally` 没有异步版本。

完整列表：`thenApplyAsync`、`thenAcceptAsync`、`thenRunAsync`、`thenComposeAsync`、`thenCombineAsync`、`handleAsync`、`whenCompleteAsync`、`exceptionally`（仅同步）。

`exceptionally` 设计为同步是因为它本身就是轻量异常恢复，不需要额外线程。

---

## Q5：CPU 密集和 IO 密集任务分别用什么线程池配置？

**A**：

| 任务类型 | 特征 | 线程数公式 | 示例 |
|---------|------|----------|------|
| CPU 密集 | 计算、解析、加密 | N+1 或 N+2 | `Runtime.getRuntime().availableProcessors() + 1` |
| IO 密集 | 网络、文件、数据库等待 | 2N 或更多 | `Runtime.getRuntime().availableProcessors() * 2` |

```java
// CPU 密集：解析 JSON、加密解密、复杂计算
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() + 1
);

// IO 密集：HTTP 调用、数据库查询、文件读写
ExecutorService ioPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 2
);

// 生产环境推荐：自定义线程池
new ThreadPoolExecutor(
    corePoolSize, maxPoolSize,
    keepAlive, TimeUnit.SECONDS,
    queue,
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

---

## Q6：thenRunAsync 和 thenAcceptAsync 的区别是什么？

**A**：

| 方法 | 输入 | 输出 | 典型场景 |
|------|------|------|---------|
| `thenAccept` | 有 T | 无（void） | 消费结果，如保存/发送 |
| `thenRun` | 无 | 无（void） | 执行与结果无关的任务 |

```java
CompletableFuture<String> cf = supplyAsync(() -> "data");

// thenAcceptAsync：接收上游结果，进行消费
cf.thenAcceptAsync(result -> {
    saveToDB(result); // 有输入参数
}, pool);

// thenRunAsync：不关心上游结果，只执行任务
cf.thenRunAsync(() -> {
    System.out.println("任务完成"); // 无输入参数
}, pool);
```

**注意**：`thenRunAsync` 不关心前一个阶段的结果，它在**前一个阶段完成时**执行，而不是**获取结果后**执行。

---

## Q7：为什么 exceptionally 没有异步版本？

**A**：因为异常处理本身应该是轻量的，不应占用线程资源：

```java
// exceptionally 的职责是异常恢复，是同步的轻量操作
.exceptionally(ex -> {
    log.error("异常", ex);
    return getFallback();
});

// 如果 exceptionally 是异步的，会导致：
// 1. 异常信息需要跨线程传递
// 2. 回调结果需要再次等待
// 3. 链式调用的异常传播变复杂
```

**替代方案**：如果异常处理本身很耗时，用 `handleAsync`：

```java
// handleAsync：统一处理，包含异常处理 + 可能的重试
.handleAsync((result, ex) -> {
    if (ex != null) {
        return retry(ex); // 异步重试
    }
    return transform(result);
}, workerPool);
```


