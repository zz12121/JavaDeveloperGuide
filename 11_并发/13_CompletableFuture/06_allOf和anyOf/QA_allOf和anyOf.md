
# allOf和anyOf

## Q1：CompletableFuture.allOf 有什么特点？如何获取各子任务的结果？

**A**：`allOf` 等待所有任务完成，返回 `CompletableFuture<Void>`，**不携带各子任务结果**。需要手动通过各 future 的 `join()` 获取：

```java
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> {
    String r1 = f1.join();
    String r2 = f2.join();
    String r3 = f3.join();
});
```

推荐封装泛型辅助方法：

```java
public static <T> CompletableFuture<List<T>> allOf(List<CompletableFuture<T>> futures) {
    return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList()));
}
```

---

## Q2：allOf 中某个子任务异常会怎样？

**A**：`allOf` **仍然会等待所有任务完成**，自身不抛异常。异常在通过 `f.join()` 获取具体子任务结果时才会暴露为 `CompletionException`。

如果需要"快速失败"（任一异常立即返回），可以用 `exceptionally` 配合 `cancel`。

---

## Q3：anyOf 的返回类型为什么是 Object？使用时注意什么？

**A**：`anyOf` 接受不同类型的 CompletableFuture，所以返回 `CompletableFuture<Object>`，使用时需要**强转**：

```java
CompletableFuture<Object> result = CompletableFuture.anyOf(stringFuture, intFuture);
String s = (String) result.join(); // 强转
```

注意：如果先完成的任务是异常完成，`anyOf` 也会异常完成。其他未完成的任务**不会自动取消**。

---

## Q4：anyOf 如果所有任务都异常会怎样？

**A**：anyOf 会以其中一个异常完成（通常是先完成的那个）：

```java
CompletableFuture<String> f1 = supplyAsync(() -> {
    Thread.sleep(100);
    throw new RuntimeException("A");
});
CompletableFuture<String> f2 = supplyAsync(() -> {
    Thread.sleep(200);
    throw new RuntimeException("B");
});

// f1 先完成且异常，anyOf 会被 f1 的异常完成
CompletableFuture.anyOf(f1, f2).join();
// 抛出 CompletionException(RuntimeException("A"))
```

---

## Q5：join() 抛出的异常为什么总是 CompletionException？

**A**：这是 JDK 的设计。所有 CF 的 join/get 方法都会将原始异常包装为 `CompletionException`：

```java
supplyAsync(() -> { throw new RuntimeException("原始"); });

try {
    cf.join();
} catch (CompletionException e) {
    e.getCause(); // RuntimeException("原始")
    e.getMessage(); // null ⚠️

    // ✅ 正确获取原始异常信息
    e.getCause().getMessage(); // "原始"
}

// ⚠️ 常见错误：直接用 getMessage()
catch (CompletionException e) {
    log.error(e.getMessage()); // 输出 null！
}
```

---

## Q6：allOf 如何实现超时控制？

**A**：配合 `orTimeout` 和 `handle`：

```java
CompletableFuture<List<User>> result = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .orTimeout(3, TimeUnit.SECONDS) // 3秒超时
    .handle((v, ex) -> {
        if (ex != null) {
            // 超时或异常，返回已完成的部分
            return futures.stream()
                .filter(f -> f.isDone() && !f.isCompletedExceptionally())
                .map(f -> (User) f.join())
                .collect(Collectors.toList());
        }
        // 正常完成，收集所有结果
        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    });
```

---

## Q7：anyOf 如何实现"多数据源竞速"？

**A**：

```java
public CompletableFuture<String> query(String key) {
    // 三个数据源同时查询
    CompletableFuture<String> cache = supplyAsync(() -> cacheService.get(key));
    CompletableFuture<String> redis = supplyAsync(() -> redisService.get(key));
    CompletableFuture<String> db = supplyAsync(() -> dbService.get(key));

    return CompletableFuture.anyOf(cache, redis, db)
        .thenApply(obj -> (String) obj)
        .whenComplete((result, ex) -> {
            // 无论哪个先返回，取消其他请求
            if (ex == null) { // 只在成功时取消
                Stream.of(cache, redis, db)
                    .filter(f -> !f.isDone())
                    .forEach(f -> f.cancel(true));
            }
        });
}
```

---

## Q8：为什么 allOf 返回 CompletableFuture<Void> 而不是结果列表？

**A**：这是 JDK 的设计选择，原因有三：

1. **异构类型问题**：`allOf(CF<String>, CF<Integer>)` 返回类型应该是？
2. **泛型擦除**：运行时无法获取具体类型信息
3. **语义一致**：allOf 语义是"等待所有完成"，不关心结果

```java
// allOf 签名
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)

// 正确获取结果
CompletableFuture<Void> all = allOf(f1, f2, f3);
all.thenRun(() -> {
    String r1 = f1.join(); // 从原始 CF 获取
    String r2 = f2.join();
    String r3 = f3.join();
});
```

---

## Q9：allOf 超时后如何返回已完成的部分结果？

**A**：

```java
public <T> CompletableFuture<List<T>> allOfWithTimeout(
        List<CompletableFuture<T>> futures,
        long timeout, TimeUnit unit) {

    return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .orTimeout(timeout, unit)
        .handle((v, ex) ->
            futures.stream()
                .filter(f -> f.isDone() && !f.isCompletedExceptionally())
                .map(CompletableFuture::join)
                .collect(Collectors.toList())
        );
}

// 使用
allOfWithTimeout(futures, 3, TimeUnit.SECONDS)
    .thenAccept(partial -> {
        log.info("获取到 {} 个结果（可能不完整）", partial.size());
        process(partial);
    });
```

---

## Q10：allOf 如何实现快速失败（fail-fast）？

**A**：

```java
// 任一异常，立即取消其他任务
CompletableFuture<Void> failFast = CompletableFuture.allOf(f1, f2, f3)
    .exceptionally(ex -> {
        futures.stream()
            .filter(f -> !f.isDone())
            .forEach(f -> f.cancel(true));
        throw (CompletionException) ex;
    });

// 配合超时
failFast.orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        futures.forEach(f -> f.cancel(true));
        return null;
    });
```

