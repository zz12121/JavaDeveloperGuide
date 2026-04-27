
# CompletableFuture实际应用场景

## Q1：CompletableFuture 在项目中最常见的应用场景是什么？

**A**：最经典的是**多服务并行调用**。

比如用户详情页需要同时查询用户信息、订单、积分，串行调用耗时是三者之和，并行调用只需最慢的那个：

```java
CompletableFuture<User> userF = supplyAsync(() -> userSvc.get(id), pool);
CompletableFuture<List<Order>> orderF = supplyAsync(() -> orderSvc.list(id), pool);
CompletableFuture<Integer> pointF = supplyAsync(() -> pointSvc.get(id), pool);

CompletableFuture.allOf(userF, orderF, pointF).join();
return new UserDetail(userF.join(), orderF.join(), pointF.join());
```

---

## Q2：服务间有依赖关系时如何编排？

**A**：用 `thenCompose` 和 `thenCombine` 构建 DAG。

- **串行依赖**（A→B）：`thenCompose`
- **并行合并**（B+C→D）：`thenCombine`
- **无依赖并行**：同时创建多个 Future

```java
CompletableFuture<A> a = supplyAsync(() -> callA());
CompletableFuture<B> b = a.thenCompose(a -> supplyAsync(() -> callB(a)));
CompletableFuture<C> c = a.thenCompose(a -> supplyAsync(() -> callC(a)));
CompletableFuture<D> d = b.thenCombine(c, (bVal, cVal) -> callD(bVal, cVal));
```

---

## Q3：如何实现超时降级？

**A**：JDK9+ 直接用 `orTimeout` + `exceptionally`：

```java
supplyAsync(() -> remoteCall(), pool)
    .orTimeout(3, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex.getCause() instanceof TimeoutException) {
            return getFromCache(); // 超时降级
        }
        throw new CompletionException(ex); // 其他异常向上抛
    });
```

JDK8 需要借助 ScheduledExecutorService 手动实现。

---

## Q4：生产环境使用 CompletableFuture 有哪些注意事项？

**A**：

1. **必须指定线程池**，不要用 commonPool（会被并行流等共享）
2. **必须设超时**，`orTimeout` 或自定义机制，防止任务永远挂住
3. **异常必须处理**，每个关键阶段加 `exceptionally` 或 `handle`
4. **线程池要隔离**，IO 密集和 CPU 密集分开
5. **控制并发数**，用 Semaphore 或固定大小线程池防止打垮下游
6. **避免在 thenApply 中做阻塞操作**，用 `thenApplyAsync` 切换到工作线程

---

## Q5：CompletableFuture 相比传统回调有什么优势？

**A**：

| 对比项 | 传统回调 | CompletableFuture |
|--------|---------|-------------------|
| 代码结构 | 嵌套回调地狱 | 链式调用，扁平清晰 |
| 异常处理 | 每层单独 try-catch | exceptionally/handle 统一处理 |
| 组合能力 | 手动管理 | thenCompose/thenCompose 内置组合 |
| 多任务聚合 | 手动 CountDownLatch | allOf/anyOf 一行搞定 |
| 线程切换 | 手动提交 | thenApplyAsync 自动提交 |
| 超时控制 | 需要额外机制 | orTimeout 原生支持 |


