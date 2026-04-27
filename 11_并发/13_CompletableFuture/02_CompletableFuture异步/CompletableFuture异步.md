
# CompletableFuture异步

## 核心结论

CompletableFuture 的异步回调方法有两种变体：**带 Async 后缀**的在指定线程池（或 commonPool）中异步执行，**不带 Async** 的在**上一个阶段完成的线程**中同步执行。理解这个区别对避免阻塞和合理分配线程至关重要。

## 深度解析

### 带 Async vs 不带 Async

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture.supplyAsync(() -> fetchData(), pool)
    .thenApply(data -> transform(data))           // ← 不带Async：在 supplyAsync 的线程中执行
    .thenApplyAsync(data -> transform(data), pool) // ← 带Async：在 pool 中新线程执行
    .thenAccept(result -> save(result));           // ← 不带Async：在 thenApplyAsync 的线程中执行
```

| 方法 | 执行线程 | 适用场景 |
|------|---------|---------|
| `thenApply(fn)` | 上一个阶段完成的线程 | 轻量操作 |
| `thenApplyAsync(fn)` | commonPool / 指定 Executor | 耗时操作或需要解耦 |
| `thenAccept(fn)` | 上一个阶段完成的线程 | 轻量消费 |
| `thenAcceptAsync(fn)` | commonPool / 指定 Executor | 耗时消费 |
| `thenRun(fn)` | 上一个阶段完成的线程 | 轻量动作 |
| `thenRunAsync(fn)` | commonPool / 指定 Executor | 耗时动作 |

### 执行线程选择策略

```java
// 场景一：CPU 密集转换 → 不需要 Async
supplyAsync(() -> loadData())
    .thenApply(data -> compute(data))  // CPU 密集但在同一线程，避免切换

// 场景二：I/O 阻塞 → 需要 Async + 独立线程池
supplyAsync(() -> loadData(), ioPool)
    .thenApplyAsync(data -> {
        db.query(data);  // I/O 阻塞，必须在新线程
        return data;
    }, ioPool)

// 场景三：混合 → 轻量用不带Async，重量用带Async
supplyAsync(() -> loadData(), ioPool)
    .thenApply(data -> data.trim())           // 轻量，同线程
    .thenApplyAsync(data -> heavyProcess(data), cpuPool)  // 重量，新线程
```

### 异步回调的线程传递

```
supplyAsync (ioPool线程T1)
    ↓ 完成
thenApply (T1 继续执行) — 同线程
    ↓ 完成
thenApplyAsync (commonPool线程T2) — 切换线程
    ↓ 完成
thenAccept (T2 继续执行) — 同线程
```

## 易错点与踩坑

### 1. 不带 Async 的方法在上游线程中执行
```java
// 关键理解：不带 Async 的回调在上游完成的线程中执行
supplyAsync(() -> {
    log.info("supply: " + Thread.currentThread().getName()); // pool-1-thread-1
    return "data";
}, ioPool)
.thenApply(data -> {
    log.info("thenApply: " + Thread.currentThread().getName()); // pool-1-thread-1 (同一线程!)
    return data.toUpperCase();
});

// 如果 ioPool 是 Tomcat 线程 → thenApply 在 Tomcat 线程中执行
// 如果在 Netty EventLoop 中 → thenApply 在 EventLoop 线程中执行
// 可能导致事件循环线程被阻塞
```

### 2. 误用 Async 导致线程过多
```java
// ❌ 每个 thenApply 都用 Async → 每步都切换线程
supplyAsync(task, pool)
    .thenApplyAsync(fn1, pool)  // 线程切换
    .thenApplyAsync(fn2, pool)  // 又切换
    .thenApplyAsync(fn3, pool); // 又切换
// 3个任务用了3个线程，实际只需要1个

// ✅ 轻量操作不需要 Async
supplyAsync(task, pool)
    .thenApply(fn1)             // 同一线程
    .thenApply(fn2)             // 同一线程
    .thenApplyAsync(fn3, pool); // 只在需要时切换
```

### 3. commonPool 线程数有限
```java
// commonPool 默认线程数 = CPU核数 - 1（最少1个）
// 如果有大量 thenApplyAsync（无参）→ 全部提交到 commonPool
// commonPool 被占满 → 其他并行操作（如 parallelStream）也受影响

// 建议：自定义线程池，不要用 commonPool（IO 场景）
```

### 4. 混合使用 Async 后的执行顺序不确定
```java
// 带 Async 后，回调在不同线程执行
// 链中的执行顺序由线程调度决定
supplyAsync(task, pool)
    .thenApplyAsync(fn1, pool)  // 可能在线程A
    .thenApply(fn2)             // 在 fn1 完成的线程中
    .thenApplyAsync(fn3, pool); // 可能在线程B
// fn2 总在 fn1 后执行，但 fn1 和 fn3 可能在不同线程中并发
```
## 关联知识点

