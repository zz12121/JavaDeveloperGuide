
# allOf和anyOf

## 核心结论

| 方法 | 语义 | 返回类型 | 完成条件 |
|------|------|---------|---------|
| `allOf(CF...)` | 等待**所有**任务完成 | `CompletableFuture<Void>` | 所有任务完成（含异常） |
| `anyOf(CF...)` | 等待**任一**任务完成 | `CompletableFuture<Object>` | 任一任务完成即返回 |

## 深度解析

### allOf — 等待所有完成

```java
CompletableFuture<String> f1 = supplyAsync(() -> callServiceA());
CompletableFuture<String> f2 = supplyAsync(() -> callServiceB());
CompletableFuture<String> f3 = supplyAsync(() -> callServiceC());

// allOf 返回 CompletableFuture<Void>，不携带各任务结果
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);

// 获取各任务结果需要手动 join
all.thenRun(() -> {
    String r1 = f1.join();
    String r2 = f2.join();
    String r3 = f3.join();
    System.out.println(r1 + r2 + r3);
}).join();
```

**注意**：
- `allOf` 本身不抛异常，即使某个子任务失败
- 需要通过 `f1.join()` 等获取结果时才会暴露异常
- 任一子任务异常，`allOf` 仍然会**等所有任务完成后**才完成

### anyOf — 任一完成即返回

```java
CompletableFuture<String> fast = supplyAsync(() -> {
    sleep(100); return "fast";
});
CompletableFuture<String> slow = supplyAsync(() -> {
    sleep(5000); return "slow";
});

// 返回最先完成的任务结果
CompletableFuture<Object> result = CompletableFuture.anyOf(fast, slow);
String value = (String) result.join(); // "fast"
```

**注意**：
- 返回类型是 `CompletableFuture<Object>`，需要**强转**
- 如果先完成的任务**抛异常**，`anyOf` 也会异常完成
- 其他未完成的任务**不会取消**，继续执行

### allOf 收集结果（推荐写法）

```java
// 泛型辅助方法
public static <T> CompletableFuture<List<T>> allOf(
        List<CompletableFuture<T>> futures) {
    CompletableFuture<Void> allDone = CompletableFuture.allOf(
        futures.toArray(new CompletableFuture[0])
    );
    return allDone.thenApply(v ->
        futures.stream()
               .map(CompletableFuture::join)
               .collect(Collectors.toList())
    );
}

// 使用
List<CompletableFuture<String>> futures = ids.stream()
    .map(id -> fetchUserAsync(id))
    .collect(Collectors.toList());

List<String> users = allOf(futures).join();
```

### 为什么 allOf 返回 Void？（核心考点）

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    allOf 返回 CompletableFuture<Void> 的原因                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  JDK 设计原因：                                                             │
│                                                                            │
│  1. 异构类型问题                                                            │
│     allOf(CF<String>, CF<Integer>, CF<User>)                               │
│     → 返回类型应该是 Void 而不是 List<Object>！                              │
│                                                                            │
│  2. 泛型擦除                                                                │
│     public static CompletableFuture<Void> allOf(CompletableFuture<?>...)   │
│     → 无法在运行时知道具体类型                                               │
│                                                                            │
│  3. 语义清晰                                                                │
│     allOf 的语义是"等待所有完成"，不关心结果                                   │
│     → 返回 Void 是合理的                                                     │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│  如果需要结果：                                                             │
│  1. 保留原始 CF 引用，通过 join() 获取                                       │
│  2. 使用 Stream 流式收集                                                    │
│  3. 封装泛型辅助方法                                                         │
└────────────────────────────────────────────────────────────────────────────┘
```

### allOf + 超时控制（优雅封装）

```java
// 超时 + 部分结果降级
public <T> CompletableFuture<List<T>> allOfWithTimeout(
        List<CompletableFuture<T>> futures,
        long timeout,
        TimeUnit unit) {

    CompletableFuture<Void> allDone = CompletableFuture.allOf(
        futures.toArray(new CompletableFuture[0])
    );

    return allDone
        .orTimeout(timeout, unit)
        .handle((v, ex) -> {
            // 超时或异常时，返回已完成的部分结果
            return futures.stream()
                .filter(f -> f.isDone() && !f.isCompletedExceptionally())
                .map(f -> {
                    try {
                        return f.join();
                    } catch (CompletionException e) {
                        return null; // 过滤异常结果
                    }
                })
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
        });
}

// 使用
List<CompletableFuture<User>> futures = userIds.stream()
    .map(id -> userService.findByIdAsync(id))
    .collect(Collectors.toList());

allOfWithTimeout(futures, 3, TimeUnit.SECONDS)
    .thenAccept(users -> {
        if (users.size() < futures.size()) {
            log.warn("部分用户加载超时，已返回 {} 个结果", users.size());
        }
        process(users);
    });
```

### allOf 超时快速失败模式

```java
// 场景：任一超时，立即取消其他任务
public CompletableFuture<List<T>> allOfFailFast(
        List<CompletableFuture<T>> futures,
        long timeout,
        TimeUnit unit) {

    CompletableFuture<List<T>> result = new CompletableFuture<>();

    // 定时器线程
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    scheduler.schedule(() -> {
        if (!result.isDone()) {
            // 取消所有未完成的任务
            futures.stream()
                .filter(f -> !f.isDone())
                .forEach(f -> f.cancel(true));
            result.completeExceptionally(
                new TimeoutException("allOf timeout after " + timeout + " " + unit)
            );
        }
    }, timeout, unit);

    // 等待所有完成
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .whenComplete((v, ex) -> {
            scheduler.shutdown();
            if (ex == null) {
                // 正常完成，收集结果
                List<T> results = futures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList());
                result.complete(results);
            } else if (!result.isDone()) {
                // 异常完成（某个子任务异常）
                result.completeExceptionally(ex);
            }
        });

    return result;
}
```

### allOf 快速失败

```java
// 默认 allOf 会等所有任务完成，即使有失败
// 如果需要"任一失败就立即返回"：
CompletableFuture<Void> allWithFailFast = CompletableFuture.allOf(f1, f2, f3)
    .exceptionally(ex -> {
        // 取消未完成的任务（仅标记，实际无法真正中断）
        Stream.of(f1, f2, f3)
            .filter(f -> !f.isDone())
            .forEach(f -> f.cancel(true));
        throw new CompletionException(ex);
    });
```

## 代码示例

### 并行调用多个服务，聚合结果

```java
public CompletableFuture<OrderDetail> getOrderDetail(String orderId) {
    CompletableFuture<Order> orderFuture = orderClient.getAsync(orderId);
    CompletableFuture<User> userFuture = userClient.getAsync(userId);
    CompletableFuture<List<Item>> itemsFuture = itemClient.getAsync(orderId);
    CompletableFuture<Address> addrFuture = addressClient.getAsync(userId);

    return CompletableFuture.allOf(orderFuture, userFuture, itemsFuture, addrFuture)
        .thenApply(v -> new OrderDetail(
            orderFuture.join(),
            userFuture.join(),
            itemsFuture.join(),
            addrFuture.join()
        ));
}
```

### 竞速调用（多数据源取最快）

```java
public CompletableFuture<String> getConfig(String key) {
    CompletableFuture<String> local = supplyAsync(() -> localCache.get(key));
    CompletableFuture<String> remote = supplyAsync(() -> remoteConfig.get(key));
    CompletableFuture<String> default_ = supplyAsync(() -> defaultValue);

    return CompletableFuture.anyOf(local, remote, default_)
        .thenApply(obj -> (String) obj);
}
```

## 易错点与踩坑

### 1. allOf 返回 CompletableFuture< Void >（核心考点）
```java
// ❌ 误以为 allOf 能返回所有结果
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.join(); // 返回 null！不是所有结果的集合

// ✅ 手动获取每个 future 的结果
CompletableFuture.allOf(f1, f2, f3).join();
String r1 = f1.join();
String r2 = f2.join();
String r3 = f3.join();

// ✅ JDK8+ 优雅写法
List<String> results = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

### 2. allOf 中一个任务异常不影响其他任务
```java
// allOf 等待所有任务完成（包括异常的）
// 一个任务异常 → allOf 的结果是异常
// 但其他正常任务仍会执行完毕

CompletableFuture<String> f1 = supplyAsync(() -> "OK");
CompletableFuture<String> f2 = supplyAsync(() -> { throw new RuntimeException("fail"); });

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
all.join(); // 抛 CompletionException（因为 f2 异常）

// f1 的结果仍然可以获取
f1.join(); // "OK" — 不受 f2 影响
f2.join(); // 抛 CompletionException
```

### 3. anyOf 返回 CompletableFuture< Object >
```java
// anyOf 的返回类型是 CompletableFuture<Object>，需要强转
CompletableFuture<String> f1 = supplyAsync(() -> "A");
CompletableFuture<Integer> f2 = supplyAsync(() -> 42);

CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);
Object result = any.join(); // 可能是 "A" 或 42，需要 instanceof 判断

// ✅ 类型安全写法
@SuppressWarnings("unchecked")
String r = (String) any.join();
```

### 4. anyOf 在所有任务都异常时抛出异常

```java
// 如果所有 future 都异常 → anyOf 也异常
CompletableFuture<String> f1 = supplyAsync(() -> { throw new RuntimeException("A"); });
CompletableFuture<String> f2 = supplyAsync(() -> { throw new RuntimeException("B"); });

CompletableFuture.anyOf(f1, f2).join(); // 抛 CompletionException
```

**anyOf 异常时的行为细节**：

```java
// 情况1：只有一个完成，其余都异常
CompletableFuture<String> f1 = supplyAsync(() -> { throw new RuntimeException("A"); });
CompletableFuture<String> f2 = supplyAsync(() -> { throw new RuntimeException("B"); });
CompletableFuture<String> f3 = supplyAsync(() -> {
    Thread.sleep(1000); // 最慢
    throw new RuntimeException("C");
});

// f1, f2 先完成且异常，f3 最慢
// anyOf 返回的是 f1 或 f2 的异常（最先完成的异常）
CompletableFuture.anyOf(f1, f2, f3).join();
// 可能抛出 RuntimeException("A") 或 RuntimeException("B")

// 情况2：所有都异常
CompletableFuture<String> allFail = CompletableFuture.anyOf(
    supplyAsync(() -> { throw new RuntimeException("A"); }),
    supplyAsync(() -> { throw new RuntimeException("B"); })
);
allFail.join(); // 抛 CompletionException，cause 取决于实现

// 情况3：泛型擦除导致的类型问题
CompletableFuture<String> sf = supplyAsync(() -> "string");
CompletableFuture<Integer> nf = supplyAsync(() -> 42);

CompletableFuture.anyOf(sf, nf).join();
// 返回值类型是 Object，可能是 String 或 Integer
// 需要 instanceof 判断
Object result = CompletableFuture.anyOf(sf, nf).join();
if (result instanceof String) {
    // ...
} else if (result instanceof Integer) {
    // ...
}
```

### 5. CompletionException 包装机制

```java
// 核心结论：join() 抛出的异常永远是 CompletionException
CompletableFuture<String> cf = supplyAsync(() -> {
    throw new RuntimeException("原始异常");
});

try {
    cf.join();
} catch (CompletionException e) {
    System.out.println(e.getCause()); // RuntimeException("原始异常")
    System.out.println(e.getMessage()); // null

    // ⚠️ getMessage() 返回 null，不会显示原始异常信息
    System.out.println(e.getCause().getMessage()); // "原始异常"
}

// 验证：直接抛 RuntimeException 时，join() 也会包装
CompletableFuture<String> cf2 = CompletableFuture
    .completedFuture("OK")
    .thenApply(s -> { throw new RuntimeException("thenApply异常"); });

try {
    cf2.join();
} catch (CompletionException e) {
    System.out.println(e.getCause()); // RuntimeException("thenApply异常")
}
```

**CompletionException 包装规则**：

```
┌──────────────────────────────────────────────────────────────┐
│                    CompletionException 包装规则                │
├──────────────────────────────────────────────────────────────┤
│  原始异常类型            │  包装方式                           │
│  ──────────────────────┼───────────────────────────────     │
│  RuntimeException      │  直接包装为 CompletionException      │
│  Error                 │  直接包装为 CompletionException      │
│  InterruptedException   │  包装为 CompletionException        │
│                         │  同时恢复中断标志                     │
│  ExecutionException    │  解包，取 cause 重新包装             │
├──────────────────────────────────────────────────────────────┤
│  ⚠️ 注意：exceptionally 中抛异常也会被包装                      │
│  exceptionally(ex -> { throw new RuntimeException(); })      │
│  → CompletionException(RuntimeException)                      │
└──────────────────────────────────────────────────────────────┘
```

### 6. allOf 返回值丢失问题

```java
// allOf 的返回值是 CompletableFuture<Void>
// 不携带任何结果！

CompletableFuture<String> f1 = supplyAsync(() -> "A");
CompletableFuture<String> f2 = supplyAsync(() -> "B");
CompletableFuture<String> f3 = supplyAsync(() -> "C");

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);

// ❌ 错误：all.join() 返回 null
Object result = all.join(); // null

// ✅ 正确：手动从每个 CF 获取结果
all.thenRun(() -> {
    System.out.println(f1.join()); // "A"
    System.out.println(f2.join()); // "B"
    System.out.println(f3.join()); // "C"
}).join();

// ✅ 推荐：Stream 流写法
List<String> results = Stream.of(f1, f2, f3)
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

### 7. anyOf 的泛型擦除问题

```java
// anyOf 签名：public static CompletableFuture<Object> anyOf(CompletableFuture<?>...)
// 返回类型是 Object，丢失了具体类型信息

CompletableFuture<String> f1 = supplyAsync(() -> "OK");
CompletableFuture<Integer> f2 = supplyAsync(() -> 123);

// ⚠️ 编译警告：unchecked conversion
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);

// ✅ 类型安全写法
@SuppressWarnings("unchecked")
CompletableFuture<String> safeString = (CompletableFuture<String>) anyOf(f1, f2);

// ✅ 更安全的泛型辅助方法
public static <T> CompletableFuture<T> anyOf(CompletableFuture<T> first,
                                               CompletableFuture<T> second) {
    return (CompletableFuture<T>) CompletableFuture.anyOf(first, second);
}
```
## 关联知识点
