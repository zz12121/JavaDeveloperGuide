
# complete 和 completeExceptionally

## 核心结论

`complete(value)` 和 `completeExceptionally(ex)` 用于**手动完成**一个 CompletableFuture，通常用于：
- 测试中模拟异步结果
- 缓存模式（有缓存直接完成）
- 取消/降级场景

## 深度解析

### complete — 手动设置正常结果

```java
CompletableFuture<String> future = new CompletableFuture<>();

// 其他线程
boolean success = future.complete("hello");
// 返回 true: 成功设置（之前未完成）
// 返回 false: 设置失败（已完成）

future.join(); // "hello"
```

**关键点**：
- 只有在 future **尚未完成**时才能设置成功
- 完成后所有等待的 `join()/get()` 立即返回
- 设置成功后 `isDone()` 返回 true

### completeExceptionally — 手动设置异常

```java
CompletableFuture<String> future = new CompletableFuture<>();

future.completeExceptionally(new RuntimeException("手动异常"));

future.join(); // 抛 CompletionException，cause 为 RuntimeException
```

### 典型场景：缓存模式

```java
public CompletableFuture<String> getData(String key) {
    CompletableFuture<String> future = new CompletableFuture<>();
    
    // 先查缓存
    String cached = cache.get(key);
    if (cached != null) {
        future.complete(cached); // 有缓存，直接完成
        return future;
    }
    
    // 无缓存，异步加载
    db.queryAsync(key).thenAccept(result -> {
        cache.put(key, result);
        future.complete(result); // 异步完成后设置
    }).exceptionally(ex -> {
        future.completeExceptionally(ex); // 传播异常
        return null;
    });
    
    return future;
}
```

### complete vs obtrudeValue

```java
// complete: 仅在未完成时设置
future.complete("value"); // 如果已完成，返回 false

// obtrudeValue: 强制覆盖，即使已完成
future.obtrudeValue("forced"); // 无论是否完成都设置

// obtrudeException: 强制设置异常
future.obtrudeException(new RuntimeException());
```

`obtrude*` 方法破坏了 CompletableFuture 的正常语义，**极少使用**。

### complete 的竞态条件

```java
CompletableFuture<String> f = new CompletableFuture<>();

// 线程A
f.complete("A");

// 线程B
f.complete("B");

// 只有先执行的会成功，后执行的返回 false
// 最终结果: "A" 或 "B"，取决于竞态
```

## 代码示例

### 测试中模拟异步结果

```java
@Test
void testOrderService() {
    // 注入 mock future
    CompletableFuture<Order> mockResult = new CompletableFuture<>();
    when(orderClient.getOrderAsync("123")).thenReturn(mockResult);
    
    // 调用被测方法
    CompletableFuture<OrderDetail> detail = orderService.getDetail("123");
    
    assertFalse(detail.isDone()); // 还未完成
    
    // 模拟异步结果到达
    mockResult.complete(new Order("123", "iPhone"));
    
    assertTrue(detail.isDone());
    assertEquals("iPhone", detail.join().getProductName());
}
```

### 超时 + 降级

```java
CompletableFuture<String> remote = httpClient.getAsync(url);

// 启动超时监控
CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(3000);
        if (!remote.isDone()) {
            remote.complete("降级数据"); // 超时后用降级值完成
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

String result = remote.join(); // 3秒内有结果就用结果，否则降级
```

## 易错点与踩坑

### 1. complete 只在未完成时生效
```java
CompletableFuture<String> cf = new CompletableFuture<>();

cf.complete("A");
cf.complete("B"); // 无效！已经完成了
cf.join(); // "A"（第一次 complete 的值）
```

### 2. completeExceptionally 的时机
```java
// completeExceptionally 常用于"外部取消"
CompletableFuture<String> cf = new CompletableFuture<>();

// 另一个线程或定时器中
scheduledExecutor.schedule(() -> {
    cf.completeExceptionally(new TimeoutException("超时"));
}, 3, TimeUnit.SECONDS);

// cf.join() → 抛 CompletionException(TimeoutException)
```

### 3. 与 cancel() 的关系
```java
// cancel(true) 内部就是调用 completeExceptionally
future.cancel(true);
// 等价于
future.completeExceptionally(new CancellationException());

// isCancelled() 判断是否被 cancel
future.isCancelled(); // true
```

### 4. obtrudeValue / obtrudeException 的危险操作
```java
// obtrudeValue 可以强制覆盖已完成的 CF 的值（包括异常状态）
// 极少使用，通常用于测试
CompletableFuture<String> cf = CompletableFuture.completedFuture("A");
cf.obtrudeValue("B"); // 强制修改为 "B"
cf.join(); // "B"

// ⚠️ 生产代码中不要使用！会破坏链式调用的异常传播
```
## 关联知识点

