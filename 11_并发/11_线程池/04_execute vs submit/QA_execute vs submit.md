
# execute vs submit

## Q1：execute 和 submit 有什么区别？

**A**：

| 维度 | execute | submit |
|------|---------|--------|
| 参数 | Runnable | Runnable / Callable |
| 返回值 | void | Future |
| 异常 | 直接抛出 | 封装在 Future |
| 来源 | Executor | ExecutorService |

**关键区别**：submit 的异常不会直接抛出，需要调用 `future.get()` 时才会抛出 `ExecutionException`。

---

## Q2：为什么 submit 的异常不会直接抛出？

**A**：submit 内部将 Runnable 包装为 `FutureTask`，FutureTask.run() 会捕获所有异常，存入 outcome 字段。调用 `future.get()` 时才检查 outcome 并抛出 `ExecutionException`。

这意味着：如果 submit 后不调用 get()，异常会被"吞掉"，不会有任何日志。

---

## Q3：什么时候用 execute，什么时候用 submit？

**A**：

- **不需要返回值，不需要关心异常** → execute（如日志异步写入）
- **需要返回值** → submit + future.get()
- **需要异常处理** → submit + try-catch future.get()
- **需要取消任务** → submit 返回 Future，可调用 cancel()

---
```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// execute：无返回值，异常直接抛出
pool.execute(() -> System.out.println("execute"));

// submit：返回 Future，异常封装在 Future 中
Future<String> future = pool.submit(() -> {
    if (Math.random() > 0.5) throw new RuntimeException("失败");
    return "success";
});

try {
    String result = future.get();  // 成功返回 "success"
} catch (ExecutionException e) {
    log.error("任务异常: " + e.getCause());  // 捕获到 RuntimeException
}
// 注意：如果不调用 future.get()，异常会被吞掉！
```

