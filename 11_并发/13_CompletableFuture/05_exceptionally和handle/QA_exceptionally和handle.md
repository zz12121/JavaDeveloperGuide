
# exceptionally和handle

## Q1：CompletableFuture 中 exceptionally 和 handle 有什么区别？

**A**：两者都能处理异常，但触发条件和行为不同：

- **`exceptionally`**：仅在**前置阶段异常时**触发，函数签名 `Function<Throwable, T>`，用于异常恢复返回默认值
- **`handle`**：**无论成功或异常都触发**，函数签名 `BiFunction<T, Throwable, U>`，既能处理异常也能转换成功结果

此外还有 **`whenComplete`**：类似 handle 但**无返回值**，不改变结果，适合日志记录。

```java
// exceptionally: 只处理异常
.future.exceptionally(ex -> "default");

// handle: 统一处理
future.handle((result, ex) -> ex != null ? "fallback" : result);

// whenComplete: 只做观察，不改结果
future.whenComplete((result, ex) -> log(result, ex));
```

---

## Q2：CompletableFuture 异常传播机制是怎样的？

**A**：异常会**跳过后续的转换操作**，沿链传播直到遇到异常处理器：

```
supplyAsync(抛异常) → thenApply(跳过) → thenAccept(跳过) → exceptionally(捕获)
```

- 被 `exceptionally` 或 `handle` 消费后，后续链**恢复正常执行**
- `whenComplete` 中如果**再次抛异常**，会**覆盖**原结果
- `handle` 的返回值会**替代**原结果

---

## Q3：实际项目中如何对 CompletableFuture 做优雅的异常处理？

**A**：推荐分层处理策略：

```java
future
    .thenApply(data -> parse(data))     // 业务转换
    .handle((result, ex) -> {
        if (ex != null) {
            log.warn("处理失败，执行降级", ex);
            return getFallback();        // 异常降级
        }
        return result;
    })
    .whenComplete((result, ex) -> {
        metrics.record(ex == null ? "ok" : "error");  // 监控埋点
    });
```

核心原则：`handle` 做降级恢复，`whenComplete` 做监控记录，职责分离。

---

## Q4：whenComplete 中抛异常会发生什么？

**A**：会**覆盖原始异常**，导致原始异常丢失！

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("原始异常"); })
    .whenComplete((result, ex) -> {
        throw new RuntimeException("whenComplete中的异常");
    });

try {
    cf.join();
} catch (CompletionException e) {
    System.out.println(e.getCause());
    // 输出：RuntimeException("whenComplete中的异常")
    // ⚠️ 原始的 RuntimeException("原始异常") 丢失了！
}
```

**原则**：whenComplete 只做日志/监控，永远不要抛异常。

---

## Q5：handle 的 function 抛异常会替换原始异常吗？

**A**：会。如果 function 抛异常，原始异常会被替换：

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("原始"); })
    .handle((result, ex) -> {
        throw new IllegalStateException("handle中抛错");
    });

// join() 抛 CompletionException(IllegalStateException("handle中抛错"))
// ❌ 原始的 RuntimeException("原始") 丢失
```

**正确做法**：

```java
// ✅ 方案1：function 中返回降级值，不抛异常
.handle((result, ex) -> {
    if (ex != null) {
        log.error("异常", ex);
        return "fallback"; // 正常返回值
    }
    return result;
});

// ✅ 方案2：必须抛异常时，包装原始异常
.handle((result, ex) -> {
    if (ex != null) {
        throw new CompletionException(ex); // 保留原始异常
    }
    return result;
});
```

---

## Q6：exceptionally 和 handle 都能处理异常，有什么场景只能用其中一个？

**A**：

| 场景 | 用 exceptionally | 用 handle |
|------|-----------------|-----------|
| 只需要恢复异常 | ✅ | 可以但不必要 |
| 需要同时处理成功和异常 | ❌ 不行 | ✅ 可以 |
| 需要基于结果做转换 | ❌ 不行 | ✅ 可以 |
| 需要无条件执行清理逻辑 | ❌ 不行 | ✅ 可以 |

```java
// 场景：无论成功失败都记录，但成功时还要转换
.handle((result, ex) -> {
    log.info("操作完成: {}", result);
    if (ex != null) {
        log.error("失败", ex);
        return getFallback();
    }
    return transform(result);
});

// exceptionally 无法做到这点，因为它在成功时不会执行
```

