
# Future vs CompletableFuture

## Q1：Future 有什么局限性？

**A**：

1. **只能阻塞获取**：`get()` 方法阻塞线程，无法注册回调
2. **无法链式调用**：获取结果后需要手动处理，不能像流式 API 一样串联
3. **无法组合**：不能将两个 Future 合并，或一个接一个执行
4. **异常处理困难**：只能通过 `get()` 抛出异常，无法精细处理
5. **无法手动完成**：不能主动设置结果值

---

## Q2：CompletableFuture 解决了哪些问题？

**A**：

| Future 的痛点 | CompletableFuture 的方案 |
|--------------|------------------------|
| 阻塞获取 | `thenAccept()` 注册回调，`getNow()` 非阻塞 |
| 无法链式 | `thenApply()` / `thenCompose()` 链式处理 |
| 无法组合 | `thenCombine()` / `allOf()` / `anyOf()` |
| 异常处理差 | `exceptionally()` / `handle()` |
| 无法手动完成 | `complete()` / `completeExceptionally()` |

---

## Q3：什么时候还用 Future？

**A**：

- 老项目兼容：`ExecutorService.submit()` 返回 `Future`
- 简单场景：只需要提交任务并等待结果，不需要链式/组合
- Spring `@Async`：默认返回 `Future`

新项目异步编程直接用 `CompletableFuture`，不需要理由。

---
```java
// Future：阻塞获取，无法链式调用
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> f1 = pool.submit(() -> fetchData("A"));
Future<String> f2 = pool.submit(() -> fetchData("B"));
String r1 = f1.get();  // 阻塞！必须等 A 完成
String r2 = f2.get();  // 阻塞！即使 B 先完成也拿不到

// CompletableFuture：非阻塞，支持链式调用和组合
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> fetchData("A"));
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchData("B"));

CompletableFuture<String> combined = cf1.thenCombine(cf2, (a, b) -> a + b);
String result = combined.join();  // 等两个都完成

// CompletableFuture.allOf：等待所有
CompletableFuture.allOf(cf1, cf2).join();
```


