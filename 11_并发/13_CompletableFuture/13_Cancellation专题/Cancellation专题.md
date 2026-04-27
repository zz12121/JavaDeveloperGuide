# Cancellation 专题

## 核心结论

`cancel()` 内部等价于 `completeExceptionally(new CancellationException())`：

```
cancel(true)  →  completeExceptionally(CancellationException)
```

- `isCancelled()` 返回 `true`
- 后续链收到 `CancellationException`（RuntimeException 子类）
- **但底层线程不会真正中断**（除非是 interruptible 任务）

## 深度解析

### cancel() 方法签名与语义

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // CompletableFuture 的 cancel 永远返回 true
    // 参数 mayInterruptIfRunning 在 CF 中被忽略！
    return internalComplete(new CancellationException()) == null;
}
```

**关键点**：
1. `mayInterruptIfRunning` 参数**被忽略**，不执行任何中断操作
2. 取消即标记状态，不涉及线程协作
3. 底层线程继续运行，只是结果被忽略

### CancellationException 的本质

```java
// CancellationException 继承自 RuntimeException
// 不是"程序错误"，是"主动放弃"的语义
public class CancellationException extends RuntimeException {
    // 无参构造，默认消息为 null
    public CancellationException() {}
    public CancellationException(String message) { super(message); }
}
```

**语义对比**：

| 异常类型 | 含义 | 传播策略 |
|---------|------|---------|
| `CancellationException` | 任务被主动取消 | 按 RuntimeException 传播 |
| `TimeoutException` | 超时未完成 | 按 RuntimeException 传播 |
| 业务异常 | 业务逻辑失败 | 按 RuntimeException 传播 |

### cancel 与 completeExceptionally 的等价性

```java
CompletableFuture<String> cf = new CompletableFuture<>();

// 方式1：直接 completeExceptionally
cf.completeExceptionally(new CancellationException("user cancelled"));

// 方式2：使用 cancel
cf.cancel(true); // 内部等价于上面

// 验证
cf.isCancelled();      // true
cf.isDone();           // true
cf.isCompletedExceptionally(); // true（注意：这个方法 JDK9+）
cf.join();             // 抛 CompletionException(CancellationException)
```

### isCancelled() vs isCompletedExceptionally()

```java
CompletableFuture<String> cf = new CompletableFuture<>();

// 初始状态
cf.isDone();                    // false
cf.isCancelled();               // false
cf.isCompletedExceptionally();  // false

// 方式1：cancel
cf.cancel(true);
cf.isCancelled();               // true ✅
cf.isCompletedExceptionally();   // true ✅（JDK9+）

// 方式2：completeExceptionally(非 Cancellation)
cf2.completeExceptionally(new RuntimeException("fail"));
cf2.isCancelled();               // false ❌
cf2.isCompletedExceptionally();  // true ✅

// 结论：只有 cancel() 会同时设置两个标志
```

### 为什么底层线程不会被中断？

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    for (int i = 0; i < 1_000_000; i++) {
        // 即使 cf.cancel(true)，这段代码仍会执行完毕
        if (Thread.currentThread().isInterrupted()) {
            System.out.println("线程被中断");
            return "interrupted";
        }
        // 业务逻辑...
    }
    return "completed";
});

// 主线程取消
cf.cancel(true);

// ⚠️ 线程不会停止，会继续执行到 return "completed"
// 但 join() 会抛 CancellationException
try {
    cf.join();
} catch (CancellationException e) {
    System.out.println("CF 被取消，收到 CancellationException");
}
```

**原因**：`cancel()` 只是标记 CF 状态为"取消"，不持有线程引用，无法主动中断。

### cancel 的实际应用场景

#### 场景1：多请求竞速取消

```java
CompletableFuture<String> query(String id) {
    // 同时查询多个数据源
    CompletableFuture<String> redis = redisClient.getAsync(id);
    CompletableFuture<String> mysql = mysqlClient.getAsync(id);
    CompletableFuture<String> es = esClient.getAsync(id);

    // anyOf 获取最快响应
    CompletableFuture<Object> winner = CompletableFuture.anyOf(redis, mysql, es);

    return winner.thenApply(obj -> (String) obj)
        .whenComplete((result, ex) -> {
            // 无论哪个先返回，取消其他请求
            if (!redis.isDone()) redis.cancel(true);
            if (!mysql.isDone()) mysql.cancel(true);
            if (!es.isDone()) es.cancel(true);
        });
}
```

#### 场景2：超时取消

```java
CompletableFuture<String> withTimeout(String taskId) {
    CompletableFuture<String> task = doTaskAsync(taskId);

    // 定时器线程
    CompletableFuture.delayedExecutor(3, TimeUnit.SECONDS)
        .execute(() -> {
            if (!task.isDone()) {
                task.cancel(true); // 标记取消
            }
        });

    return task;
}
```

#### 场景3：用户取消操作

```java
CompletableFuture<Report> generateReport(String id) {
    CompletableFuture<Report> report = reportService.generateAsync(id);

    // UI 取消按钮
    cancelButton.setOnAction(e -> {
        if (report.cancel(true)) {
            showInfo("报表生成已取消");
        }
    });

    return report;
}
```

### CancellationException 的特殊处理

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        Thread.sleep(1000);
        return "done";
    })
    .thenApply(s -> s + " processed")
    .exceptionally(ex -> {
        if (ex instanceof CancellationException) {
            return "cancelled"; // 特殊处理取消
        }
        return "error: " + ex.getMessage();
    });

// 取消
cf.cancel(true);
cf.join(); // "cancelled"
```

## 易错点与踩坑

### 1. cancel(true) 不会真正中断线程

```java
// ❌ 误以为 cancel 可以停止正在运行的线程
CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> {
    while (true) { // 无限循环
        process();
    }
});
cf.cancel(true); // 线程不会停止！

// ✅ 正确做法：在线程内部检查 isCancelled 或 isInterrupted
CompletableFuture.runAsync(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        process();
    }
});
```

### 2. isCancelled() 只对 cancel() 返回 true

```java
// ❌ 误以为手动 completeExceptionally 会使 isCancelled 为 true
cf.completeExceptionally(new CancellationException());
cf.isCancelled(); // false！

// ✅ 只有调用 cancel() 才返回 true
cf.cancel(true);
cf.isCancelled(); // true
```

### 3. 取消后无法恢复

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "data");
cf.cancel(true);

// ❌ 无法重新设置为成功
cf.complete("new data"); // 返回 false，CF 仍是取消状态
```

### 4. join() 抛出的不是 CancellationException

```java
cf.cancel(true);
try {
    cf.join();
} catch (CompletionException e) {
    // ✅ 应该这样判断
    if (e.getCause() instanceof CancellationException) {
        // 被取消了
    }
}
```

## 关联知识点

- [[complete和completeExceptionally|complete 与 completeExceptionally]]
- [[exceptionally和handle|exceptionally 与 handle]]
- [[allOf和anyOf|allOf 与 anyOf]]
