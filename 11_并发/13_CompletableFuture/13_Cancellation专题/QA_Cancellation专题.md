# Cancellation专题

## Q1：cancel() 和 completeExceptionally(new CancellationException()) 有什么区别？

**A**：功能等价，但细节不同：

```java
// cancel(true) 内部实现
return internalComplete(new CancellationException()) == null;

// 两者效果相同
cf.cancel(true);
cf.completeExceptionally(new CancellationException());

// 关键区别：isCancelled()
cf.cancel(true);
cf.isCancelled();           // true ✅

cf2.completeExceptionally(new CancellationException());
cf2.isCancelled();          // false ❌

cf3.completeExceptionally(new RuntimeException());
cf3.isCancelled();           // false ❌
```

**结论**：`cancel()` 会同时设置"已取消"标志；`completeExceptionally()` 不会设置该标志。

---

## Q2：CompletableFuture 的 cancel() 能否真正停止正在运行的线程？

**A**：不能。`cancel()` 只是标记 CF 的完成状态，不持有线程引用，无法主动中断：

```java
CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> {
    // 即使调用 cf.cancel(true)，这个线程仍会继续运行
    for (int i = 0; i < 1_000_000; i++) {
        // 除非这里检查 isInterrupted，否则不会停止
        if (Thread.currentThread().isInterrupted()) return;
        doWork(i);
    }
});

cf.cancel(true); // 只标记状态，不中断线程
```

**正确做法**：在线程内部检查中断标志，或使用 `Thread.interrupt()` 配合 `interruptible` 任务。

---

## Q3：isCancelled() 和 isCompletedExceptionally() 有什么区别？

**A**：

| 方法 | 返回 true 的情况 |
|------|-----------------|
| `isCancelled()` | 只有调用过 `cancel()` 时 |
| `isCompletedExceptionally()` | 任何通过 `completeExceptionally()` 完成时（含 cancel） |

```java
cf.cancel(true);
cf.isCancelled();               // true
cf.isCompletedExceptionally();   // true

cf2.completeExceptionally(new RuntimeException());
cf2.isCancelled();               // false
cf2.isCompletedExceptionally();  // true

cf3.completeExceptionally(new CancellationException());
cf3.isCancelled();               // false ❌（容易踩坑）
cf3.isCompletedExceptionally();  // true
```

---

## Q4：CancellationException 会被 exceptionally 捕获吗？

**A**：会。`CancellationException` 是 `RuntimeException` 的子类，会被 `exceptionally` 捕获：

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> "data")
    .thenApply(s -> {
        if (s.isEmpty()) throw new CancellationException();
        return s;
    })
    .exceptionally(ex -> {
        if (ex instanceof CancellationException) {
            return "was cancelled";
        }
        return "error: " + ex.getMessage();
    });

// 如果抛 CancellationException，结果是 "was cancelled"
```

---

## Q5：如何实现"取消即降级"的模式？

**A**：使用 `whenComplete` 统一处理：

```java
CompletableFuture<String> future = fetchData()
    .whenComplete((result, ex) -> {
        if (ex != null) {
            if (ex instanceof CancellationException) {
                log.info("任务被取消，使用降级数据");
            } else {
                log.error("任务异常", ex);
            }
        }
    })
    .exceptionally(ex -> getFallback()); // 降级值

// 如果 cancel，返回降级值
future.cancel(true);
future.join(); // fallback 值
```
