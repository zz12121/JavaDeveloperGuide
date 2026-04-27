# 异常传播专题

## Q1：异常在 CompletableFuture 链中如何传播？

**A**：

```
supplyAsync(抛异常) → thenApply(跳过) → thenAccept(跳过) → exceptionally(捕获)
```

**关键点**：
- 异常会**跳过**所有转换操作（thenApply/thenAccept/thenRun）
- 直到遇到**异常处理器**（exceptionally/handle/whenComplete）
- 异常处理器可以**捕获**异常，后续链恢复正常

---

## Q2：join() 抛出的 CompletionException 是什么？

**A**：`CompletionException` 是 Java 提供的包装异常，用于统一表示异步操作的失败：

```java
// 原始代码
throw new RuntimeException("业务异常");

// join() 抛出
throw new CompletionException(RuntimeException("业务异常"));

// 获取原始异常
try {
    cf.join();
} catch (CompletionException e) {
    Throwable cause = e.getCause(); // RuntimeException("业务异常")
}
```

**注意**：`e.getMessage()` 返回 `null`，要获取原始信息用 `e.getCause().getMessage()`。

---

## Q3：exceptionally 和 handle 都能捕获异常，区别是什么？

**A**：

| 区别 | exceptionally | handle |
|------|--------------|--------|
| 触发时机 | 仅异常 | 成功或异常都触发 |
| 函数签名 | `Function<Throwable, T>` | `BiFunction<T, Throwable, U>` |
| 成功时 | 不执行，直接跳过 | ✅ 执行 |
| 返回值 | 恢复值 | 转换后的值 |

```java
// exceptionally：只处理异常
exceptionally(ex -> "fallback"); // 成功时不执行

// handle：无论成功失败都执行
handle((result, ex) -> {
    if (ex != null) return "fallback";
    return transform(result); // 成功时做转换
});
```

---

## Q4：多层 exceptionally 抛异常会怎样？

**A**：异常会被层层替换：

```java
cf.supplyAsync(() -> { throw new RuntimeException("第1层"); })
    .exceptionally(ex -> { throw new RuntimeException("第2层"); }) // 替换
    .exceptionally(ex -> { throw new RuntimeException("第3层"); }) // 再替换
    .exceptionally(ex -> "最终恢复"); // 最终捕获

// join() 抛出的异常链：
// CompletionException
//   └─ RuntimeException("第3层")
//        └─ RuntimeException("第2层")
//             └─ RuntimeException("第1层")
```

---

## Q5：如何避免异常丢失？

**A**：三个原则：

```java
// 1. whenComplete 不要抛异常
.whenComplete((r, ex) -> {
    if (ex != null) {
        log.error("异常", ex); // ✅ 只记录，不抛
    }
});

// 2. handle 的 function 不要抛异常
.handle((r, ex) -> {
    if (ex != null) return "fallback"; // ✅ 返回值，不抛
    return r;
});

// 3. 必须抛时，保留原始异常
.exceptionally(ex -> {
    if (ex instanceof BusinessException) {
        throw (BusinessException) ex; // ✅ 保留原始类型
    }
    throw new CompletionException(ex); // ✅ 包装保留
});
```

---

## Q6： CancellationException 如何被处理？

**A**：`CancellationException` 是 `RuntimeException` 的子类，会被 `exceptionally` 捕获：

```java
cf.cancel(true); // 内部等价于 completeExceptionally(CancellationException)

cf.exceptionally(ex -> {
    if (ex instanceof CancellationException) {
        return "被取消了";
    }
    return "其他错误: " + ex.getMessage();
}).join(); // "被取消了"
```

---

## Q7：thenApply/thenAccept 中抛异常会替换原始异常吗？

**A**：会的。所有链式方法中抛出的异常都会替换当前异常：

```java
CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("原始"); })
    .thenApply(s -> { throw new IllegalStateException("thenApply中"); })
    .exceptionally(ex -> {
        System.out.println(ex.getCause()); // IllegalStateException("thenApply中")
        return "recovered";
    });
```

**异常链**：`CompletionException(IllegalStateException(RuntimeException("原始")))`
