
# get、join 和 getNow

## Q1：CompletableFuture 的 get() 和 join() 有什么区别？

**A**：

- **`get()`** 抛**受检异常**（`InterruptedException`、`ExecutionException`），必须 try-catch
- **`join()`** 抛**非受检异常**（`CompletionException`），不强制处理
- **`get()` 响应中断**，`join()` **不响应中断**（内部吞掉了 InterruptedException）

链式调用中推荐 `join()`（简洁），需要中断感知时用 `get()`。

---

## Q2：getNow 有什么用途？

**A**：`getNow(default)` 是**非阻塞**的——如果 future 已完成返回结果，否则返回默认值。适用于"有结果就用，没有就用默认值"的场景，比如轮询或提供即时反馈。

---

## Q3：为什么说 join 不响应中断是一个问题？怎么解决？

**A**：线程调用 `join()` 时即使被 `interrupt()`，也会继续阻塞直到 future 完成。这在需要优雅关闭的场景中是危险的。

解决方案：
1. 使用 `get(timeout, unit)` + catch `InterruptedException`
2. 使用 JDK9+ 的 `orTimeout()` 设置超时
3. 使用 `future.cancel(true)` 尝试中断执行线程

---
```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(1000);
    return "result";
});

// get()：阻塞，抛受检异常
try { String r = cf.get(); }
catch (InterruptedException | ExecutionException e) { }

// join()：阻塞，抛非受检异常（链式调用推荐）
String r = cf.join();

// getNow(default)：非阻塞
String r = cf.getNow("default");  // 未完成返回 "default"
```


