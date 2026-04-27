# 虚拟线程与CompletableFuture

## 核心结论

JDK 21+ 虚拟线程与 CompletableFuture 可以完美结合：

- `supplyAsync(Runnable)` 在虚拟线程中执行时，阻塞会自动 unmount
- 虚拟线程的 Carrier Thread 是 ForkJoinWorkerThread
- 使用虚拟线程可以**简化异步代码**，减少回调地狱

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    虚拟线程 + CompletableFuture 优势                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  传统异步：                                                                 │
│  supplyAsync().thenApply().thenCompose().exceptionally()...               │
│  → 链式调用复杂，线程切换开销                                                │
│                                                                            │
│  虚拟线程：                                                                 │
│  try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {            │
│      var result = doAsyncTask();  // 看起来像同步，实际异步                   │
│      var data = process(result);                                            │
│  }                                                                          │
│  → 代码简洁如同步，阻塞自动 unmount                                         │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 深度解析

### 虚拟线程中执行 CompletableFuture

```java
// 传统方式：异步链式调用
CompletableFuture<String> traditional = CompletableFuture
    .supplyAsync(() -> fetchUser(id), pool)
    .thenApplyAsync(user -> enrichUser(user), pool)
    .thenCompose(user -> saveUser(user), pool);

// 虚拟线程方式：同步写法，异步执行
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> virtual = CompletableFuture
        .supplyAsync(() -> fetchUser(id), pool); // 虚拟线程中执行

    String user = virtual.join(); // 阻塞虚拟线程，不会阻塞 Carrier Thread
    String enriched = enrichUser(user); // 继续在虚拟线程中同步执行
    String saved = saveUser(enriched);
}
```

### supplyAsync 与虚拟线程

```java
// ✅ supplyAsync 使用虚拟线程池
ExecutorService vtPool = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        // 这段代码在虚拟线程中执行
        Thread.currentThread().isVirtual(); // true
        return httpClient.get(url);
    }, vtPool);

// ✅ join() 阻塞虚拟线程，自动 unmount，不会阻塞 Carrier
String result = cf.join();

// ✅ 虚拟线程阻塞 I/O 时，Carrier Thread 可以执行其他虚拟线程
```

### thenApplyAsync 在虚拟线程中的行为

```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> cf = CompletableFuture
        .supplyAsync(() -> "data", pool)
        .thenApplyAsync(s -> {
            Thread.currentThread().isVirtual(); // true
            return process(s);
        }, pool); // 使用同一个虚拟线程池

    // ⚠️ 注意：thenApplyAsync 如果不指定 Executor，
    // 会使用 ForkJoinPool.commonPool（平台线程池），不是虚拟线程池！
}
```

### 关键陷阱：Executor 混用

```java
try (var vtPool = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> cf = CompletableFuture
        .supplyAsync(() -> "data", vtPool) // 虚拟线程 ✅

        // ❌ thenApplyAsync 无参版本使用 commonPool（平台线程）
        .thenApplyAsync(s -> process(s)) // ForkJoinPool.commonPool ⚠️

        // ✅ 指定虚拟线程池
        .thenApplyAsync(s -> process(s), vtPool) // 虚拟线程 ✅

        // ✅ 或使用 thenApply（同步，在上游线程继续）
        .thenApply(s -> process(s)); // 继承 supplyAsync 的虚拟线程 ✅
}
```

### 虚拟线程 + CompletableFuture 最佳实践

```java
// 场景：高并发 HTTP 调用
public CompletableFuture<List<User>> fetchUsers(List<String> ids) {
    try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
        // 批量创建异步任务
        List<CompletableFuture<User>> futures = ids.stream()
            .map(id -> CompletableFuture.supplyAsync(
                () -> httpClient.get("/users/" + id), pool))
            .toList();

        // allOf 等待所有完成
        return CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
}
```

### 虚拟线程 vs 传统异步：对比

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    虚拟线程 vs CompletableFuture 异步链                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  传统 CompletableFuture 链：                                                 │
│                                                                            │
│  supplyAsync(fn1)      ──线程A──► thenApply(fn2) ──线程B──►               │
│       thenApply(fn3)   ──线程C──► thenCompose(fn4) ──线程D──►              │
│       exceptionally → 返回结果                                                │
│                                                                            │
│  问题：多次线程切换，每个回调都可能有新线程                                     │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  虚拟线程 + CompletableFuture：                                              │
│                                                                            │
│  虚拟线程1: supplyAsync(fn1) → thenApply(fn2) → thenApply(fn3)            │
│                    ↑ 同一虚拟线程，自动 unmount/mount                        │
│                                                                            │
│  优势：                                                                    │
│  1. 代码看起来像同步，更易读                                                │
│  2. 无需手动管理线程池切换                                                   │
│  3. 阻塞时自动 unmount，Carrier 线程利用率高                                 │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 常见错误

### 错误1：混用虚拟线程池和 commonPool

```java
// ❌ 错误：thenApplyAsync 无参版本使用 commonPool
CompletableFuture.supplyAsync(() -> task(), vtPool)
    .thenApplyAsync(this::process) // 回到 commonPool ⚠️

// ✅ 正确：明确指定 Executor
CompletableFuture.supplyAsync(() -> task(), vtPool)
    .thenApply(this::process) // 继承上游虚拟线程 ✅
```

### 错误2：在虚拟线程中使用 ForkJoinPool.commonPool

```java
// ❌ commonPool 是平台线程池，不适合 I/O
CompletableFuture.supplyAsync(() -> httpClient.get(url))
    // 默认使用 commonPool，阻塞会影响并行流 ⚠️

// ✅ 虚拟线程池
CompletableFuture.supplyAsync(() -> httpClient.get(url), vtPool)
    // 阻塞时自动 unmount ✅
```

### 错误3：虚拟线程中误用 synchronized

```java
// ❌ JDK 21+ 虚拟线程中 synchronized 会阻塞 Carrier
CompletableFuture.supplyAsync(() -> {
    synchronized (lock) { // 阻塞 Carrier Thread ⚠️
        // 操作
    }
}, vtPool);

// ✅ 使用 ReentrantLock
CompletableFuture.supplyAsync(() -> {
    lock.lock();
    try {
        // 操作
    } finally {
        lock.unlock();
    }
}, vtPool);
```

## 适用场景

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 高并发 HTTP/DB 调用 | 虚拟线程 + CompletableFuture | 阻塞自动 unmount |
| CPU 密集计算 | ForkJoinPool + ForkJoinTask | 专用工作窃取调度 |
| 简单异步请求 | 虚拟线程 | 代码简洁如同步 |
| 需要精细控制线程 | 传统线程池 + CompletableFuture | 完全可控 |

## 关联知识点

- [[虚拟线程与ForkJoinPool|虚拟线程与 ForkJoinPool 关系]]
- [[thenApplyAsync|thenApplyAsync 线程池选择]]
- [[allOf和anyOf|allOf 批量等待]]
