
# Future vs CompletableFuture

## 核心结论

`Future` 是 JDK5 引入的异步编程接口，功能有限（阻塞获取、无法链式调用、无法组合）。`CompletableFuture` 是 JDK8 引入的增强版，支持链式调用、组合、异常处理、非阻塞，是现代 Java 异步编程的首选。

## 深度解析

### Future 局限性

```java
// Future 的使用方式（JDK5）
Future<String> future = executor.submit(() -> loadData());

// ❌ 只能阻塞等待
String result = future.get();       // 阻塞
String result = future.get(3, TimeUnit.SECONDS); // 超时阻塞

// ❌ 无法链式调用
// ❌ 无法组合多个 Future
// ❌ 无法注册回调
// ❌ 无法手动完成
```

### CompletableFuture 增强

```java
// 1. 链式调用
CompletableFuture.supplyAsync(() -> loadData())
    .thenApply(data -> transform(data))     // 转换
    .thenAccept(result -> save(result))     // 消费
    .thenRun(() -> log("done"));            // 执行

// 2. 组合
CompletableFuture<String> f1 = supplyAsync(() -> service1());
CompletableFuture<String> f2 = supplyAsync(() -> service2());
f1.thenCombine(f2, (a, b) -> a + b);       // 合并

// 3. 异常处理
supplyAsync(() -> risky())
    .exceptionally(ex -> fallback())         // 异常回退
    .handle((result, ex) -> {                // 统一处理
        return ex != null ? "error" : result;
    });

// 4. 非阻塞获取
cf.thenAccept(result -> System.out.println(result)); // 回调
String value = cf.getNow("default");                  // 立即返回
```

### 功能对比

| 特性 | Future | CompletableFuture |
|------|--------|-------------------|
| 阻塞获取 | ✅ get() | ✅ get() / join() |
| 非阻塞获取 | ❌ | ✅ getNow() / thenAccept |
| 链式调用 | ❌ | ✅ thenApply/thenCompose |
| 组合多个 | ❌ | ✅ thenCombine/allOf/anyOf |
| 异常处理 | ❌ | ✅ exceptionally/handle |
| 手动完成 | ❌ | ✅ complete() |
| 超时控制 | ❌（get 超时） | ✅ orTimeout() |
| 多回调注册 | ❌ | ✅ 可注册多个 |

### 迁移建议

```java
// 旧代码（Future）
Future<String> f = executor.submit(callable);
String result = f.get();

// 新代码（CompletableFuture）
String result = CompletableFuture.supplyAsync(callable, executor).join();
// 或者继续链式调用
```

## 关联知识点
