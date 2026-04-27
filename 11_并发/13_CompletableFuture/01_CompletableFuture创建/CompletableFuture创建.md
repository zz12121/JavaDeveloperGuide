
# CompletableFuture创建

## 核心结论

CompletableFuture 是 JDK8 引入的**异步编程工具**，支持链式调用、组合、异常处理。创建方式：
- `supplyAsync(Supplier)` — 有返回值的异步任务
- `runAsync(Runnable)` — 无返回值的异步任务
- 可指定自定义 Executor，默认使用 ForkJoinPool.commonPool

## 深度解析

### 四种创建方式

```java
// 1. 有返回值，默认 commonPool
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

// 2. 有返回值，自定义线程池
ExecutorService pool = Executors.newFixedThreadPool(4);
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
}, pool);

// 3. 无返回值，默认 commonPool
CompletableFuture<Void> f3 = CompletableFuture.runAsync(() -> {
    System.out.println("Running");
});

// 4. 无返回值，自定义线程池
CompletableFuture<Void> f4 = CompletableFuture.runAsync(() -> {
    System.out.println("Running");
}, pool);
```

### supplyAsync vs runAsync

| 特性 | supplyAsync | runAsync |
|------|------------|----------|
| 参数 | Supplier\<T\> | Runnable |
| 返回值 | CompletableFuture\<T\> | CompletableFuture\<Void\> |
| 适用场景 | 需要返回结果的异步任务 | 只需执行不需返回 |

### 线程池选择

```java
// ❌ I/O 任务用 commonPool — 会阻塞其他并行操作
CompletableFuture<String> bad = CompletableFuture.supplyAsync(() -> {
    return httpClient.get("http://api.example.com");  // 阻塞！
});

// ✅ I/O 任务用独立线程池
ExecutorService ioPool = Executors.newFixedThreadPool(10);
CompletableFuture<String> good = CompletableFuture.supplyAsync(() -> {
    return httpClient.get("http://api.example.com");
}, ioPool);
```

### 其他创建方式

```java
// 创建已完成的 CF（测试常用）
CompletableFuture<String> completed = CompletableFuture.completedFuture("done");

// 创建未完成的 CF（手动控制完成）
CompletableFuture<String> manual = new CompletableFuture<>();
manual.complete("result");       // 正常完成
manual.completeExceptionally(new RuntimeException("fail"));  // 异常完成
```

## 易错点与踩坑

### 1. IO 任务不要用 commonPool
```java
// ❌ IO 阻塞会占用 commonPool 线程
// commonPool 线程数 = CPU核数-1，被 IO 阻塞后其他并行操作也卡住
CompletableFuture.supplyAsync(() -> httpClient.get("http://api.example.com"));
// parallelStream() 等也依赖 commonPool → 互相影响

// ✅ IO 任务用独立线程池
ExecutorService ioPool = Executors.newFixedThreadPool(20);
CompletableFuture.supplyAsync(() -> httpClient.get("http://api.example.com"), ioPool);
```

### 2. completedFuture 的用途容易忽略
```java
// 单元测试中模拟异步结果
CompletableFuture<String> mock = CompletableFuture.completedFuture("test-data");
// 不需要真正执行异步操作，直接返回已完成的结果

// 与 new CompletableFuture() 对比：
// completedFuture → 已完成，立即可用
// new CompletableFuture<>() → 未完成，需要手动 complete
```

### 3. new CompletableFuture() 忘记 complete
```java
// ❌ 创建后忘记调用 complete 或 completeExceptionally
CompletableFuture<String> cf = new CompletableFuture<>();
// 任何调用 cf.get() 或 cf.join() 的线程会永远阻塞！

// ✅ 确保在所有路径上都有完成操作
try {
    cf.complete(doSomething());
} catch (Exception e) {
    cf.completeExceptionally(e);
}
```

### 4. supplyAsync 的异常不会自动传播
```java
// supplyAsync 中抛异常 → CF 进入异常状态
// 但如果没有链式处理（exceptionally/handle）→ 异常直到 get/join 才暴露
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("fail");
});
// cf.get() → ExecutionException
// cf.join() → CompletionException
```
## 关联知识点

