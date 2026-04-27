
# 超时处理

## Q1：CompletableFuture 如何实现超时控制？

**A**：JDK9+ 提供两种方式：

- **`orTimeout(duration, unit)`**：超时后异常完成，抛 `TimeoutException`
- **`completeOnTimeout(value, duration, unit)`**：超时后正常完成，返回默认值

JDK8 中需要自行实现：用 `ScheduledExecutorService` 延迟调用 `complete(defaultValue)`。

---

## Q2：orTimeout 超时后底层任务会停止执行吗？

**A**：**不会**。`orTimeout` 只影响 CompletableFuture 的完成状态，不会取消或中断底层执行线程。这是一个常见的误解。

如果需要在超时时真正停止任务，需要额外处理（如 `Future.cancel(true)` 配合线程中断检查）。

---

## Q3：实际项目中超时处理如何与降级结合？

**A**：典型模式是 `orTimeout` + `exceptionally`：

```java
future.orTimeout(3, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex.getCause() instanceof TimeoutException) {
            return getFallback();  // 超时降级
        }
        throw new CompletionException(ex);  // 其他异常继续传播
    });
```

