
# get、join 和 getNow

## 核心结论

| 方法 | 阻塞 | 异常类型 | 超时支持 | 返回 |
|------|------|---------|---------|------|
| `get()` | ✅ 阻塞 | `InterruptedException` + `ExecutionException`（受检） | ❌ | 结果值 |
| `get(timeout, unit)` | ✅ 限时阻塞 | `TimeoutException`（超时） + 受检异常 | ✅ | 结果值 |
| `join()` | ✅ 阻塞 | `CompletionException`（非受检） | ❌ | 结果值 |
| `getNow(default)` | ❌ 不阻塞 | 无 | ❌ | 完成值或默认值 |

## 深度解析

### get() — 受检异常

```java
try {
    String result = future.get(); // 阻塞等待
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断状态
} catch (ExecutionException e) {
    Throwable cause = e.getCause(); // 获取实际异常
}
```

**特点**：
- 抛出**受检异常**，必须 try-catch 或 throws
- `InterruptedException`：等待线程被中断
- `ExecutionException`：异步任务内部抛异常，通过 `getCause()` 获取原始异常

### join() — 非受检异常

```java
String result = future.join(); // 阻塞等待，无需 try-catch
// 异常时抛 CompletionException（RuntimeException）
```

**特点**：
- 抛出 `CompletionException`（**非受检**），不强制处理
- 内部异常包装在 `CompletionException.getCause()` 中
- **推荐在链式调用中使用**，代码更简洁
- **不响应中断**（内部吞掉了 InterruptedException）

### getNow(default) — 非阻塞

```java
// 如果已完成，返回结果；否则返回默认值
String result = future.getNow("默认值");
```

**特点**：
- **立即返回**，不阻塞
- 适用于"有就用，没有就用默认值"的场景
- 不会改变 CompletableFuture 的状态

### 三者对比

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        sleep(2000);
        return "hello";
    });

// get: 阻塞2秒，抛受检异常
String r1 = future.get();

// join: 阻塞2秒，抛非受检异常
String r2 = future.join();

// getNow: 立即返回 "default"，不等待
String r3 = future.getNow("default");

// get(1, SECONDS): 等待1秒后超时抛 TimeoutException
String r4 = future.get(1, TimeUnit.SECONDS);
```

### join 不响应中断的问题

```java
Thread t = new Thread(() -> {
    try {
        future.join(); // 即使线程被 interrupt，也会继续等待
    } catch (CancellationException e) {
        // 只有在 future 被 cancel 时才会抛
    }
});
t.start();
t.interrupt(); // join 不会因此抛异常！
```

**解决方案**：使用 `get()` 或 `orTimeout()`。

## 代码示例

### 推荐：链式场景用 join

```java
// 链式调用中用 join，代码简洁
List<String> results = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

### 推荐：需要中断响应用 get

```java
// 需要响应中断时用 get
try {
    String result = future.get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    log.warn("超时");
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    log.warn("被中断");
}
```

### 非阻塞轮询模式

```java
while (!future.isDone()) {
    String current = future.getNow("loading...");
    System.out.println(current);
    Thread.sleep(500);
}
System.out.println(future.join()); // 最终结果
```

## 易错点与踩坑

### 1. join() 不响应中断（核心考点）
```java
// join() 内部吞掉了 InterruptedException
Thread t = new Thread(() -> future.join());
t.start();
t.interrupt(); // join() 不会抛异常，继续等待！

// ✅ 需要响应中断时用 get()
try {
    future.get(5, TimeUnit.SECONDS);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断标志
    // 处理中断逻辑
}
```

### 2. get() 的 ExecutionException 需要解包
```java
// get() 抛出的是 ExecutionException，实际异常在 getCause() 中
try {
    future.get();
} catch (ExecutionException e) {
    Throwable cause = e.getCause(); // 原始异常
    if (cause instanceof RuntimeException) {
        // 处理
    }
}
// 不要直接处理 ExecutionException，要解包
```

### 3. getNow 的"快照"语义
```java
// getNow(default) 只是一个快照，不是"注册回调"
// 调用后立即返回，不会等待
String now = future.getNow("default");
// 如果 future 已完成 → 返回结果
// 如果未完成 → 返回 "default"（但不注册任何回调）

// ❌ 不要用 getNow 做轮询等待
while (future.getNow(null) == null) { /* 忙等待，浪费 CPU */ }
// ✅ 用 join 或 get 等待
```

### 4. 链式调用中 get vs join 的选择
```java
// ✅ 链式调用中推荐 join（简洁）
List<String> results = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());

// ✅ 需要超时用 get
future.get(3, TimeUnit.SECONDS);

// ❌ 不要在 thenApply 等回调中使用 get（阻塞回调线程）
.thenApply(data -> {
    future2.get(); // 阻塞了回调线程！违背异步初衷
});
```
## 关联知识点

