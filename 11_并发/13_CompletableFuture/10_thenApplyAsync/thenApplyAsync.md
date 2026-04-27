
# thenApplyAsync

## 核心结论

`thenApplyAsync` 是 `thenApply` 的**异步版本**，转换任务在**指定线程池**（或默认 ForkJoinPool.commonPool）中执行，而不是在触发它的线程中执行。

## 深度解析

### thenApply vs thenApplyAsync

| 特性 | thenApply | thenApplyAsync |
|------|-----------|----------------|
| 执行线程 | 上游任务完成的线程 | 独立线程池中的线程 |
| 线程切换 | 不切换 | 切换到新线程 |
| 适用场景 | 轻量转换，无需额外线程 | 耗时操作，需要独立线程 |
| 无参版本 | 无 | 使用 ForkJoinPool.commonPool |
| 有参版本 | 无 | 可指定自定义 Executor |

### 执行模型

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    同步 vs 异步执行流程对比                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  【同步版本 thenApply】                                                     │
│                                                                            │
│  线程A: [supplyAsync] ──完成──> [thenApply] ──同步执行──> 结果              │
│         ◄──────────────── 同一线程 ──────────────────────────────────►    │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  【异步版本 thenApplyAsync】                                                │
│                                                                            │
│  线程A: [supplyAsync] ──完成──> [提交到线程池]                              │
│                                        │                                  │
│                                        ▼                                  │
│                                   ┌─────────┐                             │
│                                   │ 队列    │                             │
│                                   └────┬────┘                             │
│                                        │                                  │
│                    ┌─────────────────┬┴─┬─────────────────┐                │
│                    ▼                 ▼                 ▼                │
│               线程B执行          线程C执行         线程D执行              │
│               thenApplyAsync    thenApplyAsync    thenApplyAsync         │
│                                                                            │
│  注意：回调结果可能在不同线程中，但数据依赖保证执行顺序                          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 异步三兄弟：thenApply / thenAccept / thenRun

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         异步方法三剑客                                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  方法              │ 函数签名                   │ 用途                    │
│  ─────────────────┼────────────────────────────┼───────────────────        │
│  thenApply        │ Function<T, U>            │ 转换，有返回值              │
│  thenApplyAsync  │ Function<T, U> + Executor  │ 异步转换                   │
│                                                                            │
│  thenAccept       │ Consumer<T>                │ 消费，无返回值              │
│  thenAcceptAsync │ Consumer<T> + Executor     │ 异步消费                   │
│                                                                            │
│  thenRun          │ Runnable                   │ 执行任务，无输入无输出      │
│  thenRunAsync    │ Runnable + Executor        │ 异步执行任务               │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  thenApplyAsync：输入 T → 输出 U → 返回 CompletableFuture<U>                 │
│  thenAcceptAsync：输入 T → 输出 void → 返回 CompletableFuture<Void>          │
│  thenRunAsync：   无输入 → 输出 void → 返回 CompletableFuture<Void>          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 关键方法

```java
// 无参：使用 ForkJoinPool.commonPool
CompletableFuture<U> thenApplyAsync(Function<T, U> fn)

// 有参：使用指定 Executor
CompletableFuture<U> thenApplyAsync(Function<T, U> fn, Executor executor)
```

### 异步系列方法全景

| 同步方法 | 异步方法（默认池） | 异步方法（指定池） |
|---------|-------------------|-------------------|
| thenApply | thenApplyAsync(fn) | thenApplyAsync(fn, exec) |
| thenAccept | thenAcceptAsync(con) | thenAcceptAsync(con, exec) |
| thenRun | thenRunAsync(run) | thenRunAsync(run, exec) |
| thenCompose | thenComposeAsync(fn) | thenComposeAsync(fn, exec) |
| thenCombine | thenCombineAsync(other, fn) | thenCombineAsync(other, fn, exec) |
| handle | handleAsync(fn) | handleAsync(fn, exec) |
| whenComplete | whenCompleteAsync(fn) | whenCompleteAsync(fn, exec) |
| exceptionally | 无异步版本 | 无异步版本 |

## 代码示例

```java
ExecutorService myPool = Executors.newFixedThreadPool(3);

CompletableFuture.supplyAsync(() -> {
    log.info("supplyAsync: {}", Thread.currentThread().getName());
    return 100;
}, myPool)
.thenApply(value -> {
    // 在 supplyAsync 的线程中执行
    log.info("thenApply: {}", Thread.currentThread().getName());
    return value * 2;
})
.thenApplyAsync(value -> {
    // 在 ForkJoinPool.commonPool 中执行
    log.info("thenApplyAsync: {}", Thread.currentThread().getName());
    return value + 50;
})
.thenApplyAsync(value -> {
    // 在指定线程池中执行
    log.info("thenApplyAsync+exec: {}", Thread.currentThread().getName());
    return value;
}, myPool)
.join(); // 250
```

### 实际场景：避免阻塞事件循环

```java
// Netty/事件驱动框架中，不能阻塞 IO 线程
eventLoop.submit(() -> {
    // 在事件循环线程中
    return readRequest();
}).thenApplyAsync(request -> {
    // 耗时计算，不能在事件循环线程中执行
    return heavyComputation(request);
}, workerPool)  // 提交到工作线程池
.thenAcceptAsync(result -> {
    // 写回结果，回到事件循环线程
    eventLoop.execute(() -> writeResponse(result));
}, eventLoopExecutor);
```

## 线程池选择策略

### 场景化选择

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         线程池选择决策树                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                         需要选择线程池                                        │
│                              │                                              │
│              ┌───────────────┼───────────────┐                            │
│              ▼               ▼               ▼                            │
│       CPU 密集型        IO 密集型        混合场景                            │
│       (计算/解析)       (网络/文件)                                      │
│              │               │               │                            │
│              ▼               ▼               ▼                            │
│     核心数+1 线程      2N 或更多线程     分离池策略                          │
│     (避免上下文切换)   (充分利用等待)    CPU池 + IO池                        │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 线程池配置参考

```java
// CPU 密集型：核心数 + 1（留一个备用于 GC/系统）
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() + 1
);

// IO 密集型：2N（充分利用 IO 等待时间）
ExecutorService ioPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 2
);

// 生产推荐：自定义线程池
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setName("biz-pool-" + t.getId());
    t.setDaemon(false);
    return t;
};
ExecutorService bizPool = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    factory,
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略：调用者执行
);
```

### 异步链中的线程池切换模式

```java
// 典型模式：IO 查询 → CPU 计算 → IO 写入
public CompletableFuture<Result> process(String id) {
    return CompletableFuture
        // Step 1: IO 查询（使用 IO 线程池）
        .supplyAsync(() -> queryFromDB(id), ioPool)

        // Step 2: CPU 计算（切换到 CPU 线程池）
        .thenApplyAsync(data -> heavyComputation(data), cpuPool)

        // Step 3: 继续在 CPU 线程池（同步版本）
        .thenApply(data -> aggregate(data))

        // Step 4: IO 写入（切换回 IO 线程池）
        .thenAcceptAsync(result -> saveToDB(result), ioPool);
}
```

## 易错点与踩坑

### 1. thenApplyAsync 无参版本使用 commonPool
```java
// ❌ 无参 Async 用 commonPool
.thenApplyAsync(data -> heavyProcess(data))  // commonPool，IO 阻塞会影响 parallelStream

// ✅ 指定线程池
.thenApplyAsync(data -> heavyProcess(data), cpuPool)
```

### 2. 在事件循环/IO线程中使用 thenApplyAsync
```java
// ❌ 在 Netty EventLoop 中阻塞
channel.writeAndFlush(request)
    .thenApply(response -> {
        return db.query(response.getId()); // 阻塞了 EventLoop 线程！
    });

// ✅ 切到工作线程
channel.writeAndFlush(request)
    .thenApplyAsync(response -> {
        return db.query(response.getId());
    }, workerPool)  // 切到工作线程池
    .thenAcceptAsync(result -> {
        channel.writeAndFlush(result);  // 回到 EventLoop 写回
    }, eventLoopExecutor);
```

### 3. 异步回调中的线程上下文丢失
```java
// 主线程设置了 ThreadLocal
ThreadLocal<String> context = new ThreadLocal<>();
context.set("trace-123");

supplyAsync(() -> fetchData(), pool)
    .thenApplyAsync(data -> {
        // context.get() → null！新线程没有 ThreadLocal
        return data;
    }, pool);

// ✅ 手动传递上下文
String traceId = context.get();
supplyAsync(() -> fetchData(), pool)
    .thenApplyAsync(data -> {
        context.set(traceId); // 恢复上下文
        try {
            return process(data);
        } finally {
            context.remove(); // 清理，防止线程池复用时泄漏
        }
    }, pool);
```

### 4. thenApplyAsync 的执行顺序
```java
// thenApplyAsync 不保证顺序执行（如果用不同线程）
// 但同一个 CF 链中，thenApplyAsync 的提交是有序的
// 实际执行顺序取决于线程池调度
supplyAsync(() -> 1, pool)
    .thenApplyAsync(n -> n + 1, pool)  // 可能在线程A
    .thenApplyAsync(n -> n * 2, pool); // 必须在前一步完成后执行
// 虽然线程不同，但数据依赖保证顺序
```
## 关联知识点

