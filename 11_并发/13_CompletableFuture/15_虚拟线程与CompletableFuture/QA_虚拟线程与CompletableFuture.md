# 虚拟线程与CompletableFuture

## Q1：虚拟线程中可以使用 CompletableFuture 吗？

**A**：可以，而且是非常好的组合：

```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> cf = CompletableFuture
        .supplyAsync(() -> httpClient.get(url), pool);

    // join() 阻塞虚拟线程，不会阻塞 Carrier Thread
    String result = cf.join();
}
```

**优势**：
- 代码看起来像同步代码
- 阻塞时自动 unmount，Carrier 线程可执行其他虚拟线程
- 无需复杂的链式回调

---

## Q2：thenApplyAsync 无参版本在虚拟线程中有什么问题？

**A**：`thenApplyAsync()` 无参版本使用 `ForkJoinPool.commonPool`（平台线程池），不是虚拟线程池：

```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture.supplyAsync(() -> task(), pool)
        .thenApplyAsync(fn) // ⚠️ 使用 commonPool，回到平台线程
        .thenApplyAsync(fn, pool) // ✅ 指定虚拟线程池
        .thenApply(fn) // ✅ 同步版本，继承上游虚拟线程
}
```

---

## Q3：虚拟线程中 join() 和 get() 会阻塞 Carrier 线程吗？

**A**：不会。`join()` / `get()` 在虚拟线程中调用时，会触发 **unmount**，Carrier 线程可以被其他虚拟线程使用：

```java
// 虚拟线程中
CompletableFuture<String> cf = supplyAsync(() -> fetchData(), vtPool);
String result = cf.join(); // 阻塞虚拟线程，Carrier unmount
// Carrier 线程可以执行其他虚拟线程的任务
```

**对比**：
- 平台线程中 `join()`：线程阻塞，空等
- 虚拟线程中 `join()`：虚拟线程阻塞，Carrier unmount

---

## Q4：为什么虚拟线程中要避免 synchronized？

**A**：JDK 21+ 虚拟线程中使用 `synchronized` 会阻塞 Carrier 线程，影响所有共享该 Carrier 的虚拟线程：

```java
// ❌ 危险
synchronized (lock) {
    // 阻塞 Carrier Thread！
}

// ✅ 安全
lock.lock();
try {
    // 操作
} finally {
    lock.unlock();
}
```

**原因**：`synchronized` 的膨胀锁定（inflated locking）会直接阻塞 OS 线程，而 `ReentrantLock` 支持 interruptibly。

---

## Q5：虚拟线程 + CompletableFuture vs 纯 CompletableFuture 链，怎么选？

**A**：

| 因素 | 虚拟线程 + CF | 纯 CF 链 |
|------|--------------|---------|
| 代码可读性 | 高（同步写法） | 中（链式回调） |
| 线程切换 | 少（同一虚拟线程） | 多（每个回调可能新线程） |
| 适用场景 | I/O 密集 | 所有场景 |
| 学习成本 | 低 | 中 |
| 精细控制 | 弱 | 强 |

**建议**：
- I/O 密集（HTTP/DB）→ 虚拟线程 + CF
- CPU 密集或需要精细控制 → 纯 CF 链或 ForkJoinPool

---

## Q6：虚拟线程池中使用 allOf/anyOf 有什么注意事项？

**A**：

```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    List<CompletableFuture<User>> futures = ids.stream()
        .map(id -> CompletableFuture.supplyAsync(
            () -> fetchUser(id), pool))
        .toList();

    // ✅ allOf 等待所有虚拟线程完成
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenAccept(v -> {
            List<User> users = futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
        });
}
```

**注意**：allOf 返回 `CompletableFuture<Void>`，需要从原始 futures 获取结果。
