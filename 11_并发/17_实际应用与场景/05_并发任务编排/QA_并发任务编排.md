---
tags: [Java并发, CompletableFuture, 任务编排, 异步编程]
module: 17_实际应用与场景
chapter: 05_并发任务编排
---

# 并发任务编排

## Q1：如何实现多个异步任务并行执行并等待全部完成？

**A**：

使用 `CompletableFuture.allOf`：

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchData1(), executor);
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchData2(), executor);
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> fetchData3(), executor);

CompletableFuture.allOf(f1, f2, f3)
    .get(5, TimeUnit.SECONDS);  // 阻塞等待，超时抛 TimeoutException

// allOf 完成后获取各自结果
String r1 = f1.join();
String r2 = f2.join();
String r3 = f3.join();
```

或用 `CountDownLatch`：

```java
CountDownLatch latch = new CountDownLatch(3);
executor.submit(() -> { task1(); latch.countDown(); });
executor.submit(() -> { task2(); latch.countDown(); });
executor.submit(() -> { task3(); latch.countDown(); });
latch.await(5, TimeUnit.SECONDS);
```

**两者区别**：`CompletableFuture.allOf` 可链式处理结果和异常；`CountDownLatch` 无法直接获取子任务返回值。

---

## Q2：thenApply 和 thenCompose 有什么区别？

**A**：

核心区别：**是否需要扁平化**。

| 方法 | 函数类型 | 返回值 | 适用场景 |
|------|---------|--------|---------|
| `thenApply(f)` | `T → U` | 直接包装为 `CF<U>` | 同步结果变换，不涉及异步调用 |
| `thenCompose(f)` | `T → CF<U>` | 扁平为 `CF<U>`（而非 `CF<CF<U>>`） | 下一步本身也是异步操作 |

```java
// thenApply：结果变换（不涉及异步）
CompletableFuture<String> name = CompletableFuture
    .supplyAsync(() -> 1L)
    .thenApply(id -> "user_" + id);   // Long → String，简单变换

// thenCompose：异步调用链（避免嵌套地狱）
CompletableFuture<Order> order = CompletableFuture
    .supplyAsync(() -> getUserId())          // CF<Long>
    .thenCompose(id -> getOrderAsync(id));   // 扁平成 CF<Order>

// ❌ 错误写法：用 thenApply 调用异步方法 → 得到 CF<CF<Order>>，需要再 .join()
CompletableFuture<CompletableFuture<Order>> nested = CompletableFuture
    .supplyAsync(() -> getUserId())
    .thenApply(id -> getOrderAsync(id));     // CF<CF<Order>> ← 嵌套！
```

**记忆口诀**：如果 lambda 返回的是 `CompletableFuture`，就用 `thenCompose`；否则用 `thenApply`。

---

## Q3：CountDownLatch 和 CompletableFuture.allOf 有什么区别？

**A**：

| 维度 | CountDownLatch | CompletableFuture.allOf |
|------|---------------|------------------------|
| 返回值 | 无（void） | 可获取每个任务的结果 |
| 异常处理 | 需在子线程内 try-catch | 可统一 `.exceptionally()`/`.handle()` |
| 链式调用 | ❌ 不支持 | ✅ 可链式 `.thenApply()` 等 |
| 超时控制 | `await(timeout, unit)` | `.orTimeout()` / `.get(timeout, unit)` |
| 风格 | 命令式 | 函数式 |

推荐优先用 `CompletableFuture`，更灵活、更现代。`CountDownLatch` 适合需要精确控制计数的场景（如多阶段等待）。

---

## Q4：CompletableFuture 如何处理异常？exceptionally、handle、whenComplete 怎么选？

**A**：

三种方式各有侧重：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (error) throw new RuntimeException("调用失败");
    return "数据";
});

// 1. exceptionally：只捕获异常，返回兜底值；正常值直接透传
future.exceptionally(ex -> {
    log.error("异常: {}", ex.getMessage());
    return "默认数据";  // 异常时替换为默认值
});

// 2. handle：无论正常/异常都执行，可以同时做变换
future.handle((result, ex) -> {
    if (ex != null) return "默认数据";
    return result.toUpperCase();  // 对正常结果做处理
});

// 3. whenComplete：副作用，不改变结果（观察用）
future.whenComplete((result, ex) -> {
    // result 和 ex 只能读，返回值无效
    metrics.record(ex != null ? "error" : "success");
});
```

**选择口诀**：
- 只处理异常 + 兜底值 → `exceptionally`
- 正常异常都要处理 + 需要变换结果 → `handle`
- 只做日志/监控，不改变结果 → `whenComplete`

---

## Q5：如何实现任务 A 完成后再执行任务 B，且 B 需要调用另一个异步接口？

**A**：

使用 `thenCompose` 实现异步串行依赖：

```java
// 场景：先获取用户ID → 再异步查询该用户的订单列表
CompletableFuture<List<Order>> orders = CompletableFuture
    .supplyAsync(() -> userService.getUserId(name), executor)   // 步骤1：查用户ID
    .thenCompose(userId ->
        CompletableFuture.supplyAsync(                           // 步骤2：查订单（异步）
            () -> orderService.listByUser(userId), executor
        )
    )
    .exceptionally(ex -> {
        log.error("查询失败", ex);
        return Collections.emptyList();
    });

List<Order> result = orders.get(3, TimeUnit.SECONDS);
```

**为什么不用 thenApply**：`orderService.listByUser` 本身是异步方法，返回 `CF<List<Order>>`，用 `thenApply` 会得到 `CF<CF<List<Order>>>` 嵌套，必须用 `thenCompose` 扁平化。

---

## Q6：如何合并两个并行异步任务的结果？

**A**：

使用 `thenCombine` 合并两个独立 Future 的结果：

```java
// 场景：并行查商品价格 + 库存，合并展示
CompletableFuture<Double> priceFuture =
    CompletableFuture.supplyAsync(() -> priceService.getPrice(skuId), executor);

CompletableFuture<Integer> stockFuture =
    CompletableFuture.supplyAsync(() -> stockService.getStock(skuId), executor);

// thenCombine：等两个都完成，将结果合并
CompletableFuture<ProductVO> vo = priceFuture.thenCombine(
    stockFuture,
    (price, stock) -> new ProductVO(skuId, price, stock)
);

// thenCombine vs allOf：
// - thenCombine：合并两个，直接拿到类型安全的两个结果
// - allOf：等待N个，需手动 join 各Future获取结果
```

```java
// 等待多个 Future（>2个）时用 allOf + join
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> {
    String r1 = f1.join();
    String r2 = f2.join();
    String r3 = f3.join();
    merge(r1, r2, r3);
});
```

---

## Q7：生产环境中 CompletableFuture 有哪些常见陷阱？

**A**：

**陷阱1：不指定自定义线程池，耗尽 ForkJoinPool 公共池**

```java
// ❌ 默认使用 ForkJoinPool.commonPool()，CPU密集任务/IO阻塞会卡死其他任务
CompletableFuture.supplyAsync(() -> callRemoteService());

// ✅ 为不同类型任务指定独立线程池
private static final Executor IO_POOL = new ThreadPoolExecutor(
    20, 100, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000)
);
CompletableFuture.supplyAsync(() -> callRemoteService(), IO_POOL);
```

**陷阱2：get() 不设超时，导致线程永久阻塞**

```java
// ❌ 没有超时，下游服务故障会hang住调用线程
String result = future.get();

// ✅ 必须设置超时
String result = future.get(3, TimeUnit.SECONDS);

// ✅ 或用 orTimeout（Java 9+）
future.orTimeout(3, TimeUnit.SECONDS).exceptionally(ex -> "默认值");
```

**陷阱3：异常被吞掉**

```java
// ❌ 异常在 thenApply 内部抛出但没有 exceptionally，外部不知道失败
future.thenApply(r -> dangerousOp(r));  // 异常被包在 CF 里，调用方不 get() 就察觉不到

// ✅ 任何 CF 链末尾都要有异常处理
future.thenApply(r -> dangerousOp(r))
      .exceptionally(ex -> { log.error("失败", ex); return fallback(); });
```

**陷阱4：thenApply vs thenApplyAsync 线程切换问题**

```java
// thenApply：在触发完成的线程中执行（可能是主线程）
// thenApplyAsync：提交到指定线程池执行（推荐IO/耗时操作）
future.thenApplyAsync(result -> heavyProcess(result), PROCESS_POOL);
```
