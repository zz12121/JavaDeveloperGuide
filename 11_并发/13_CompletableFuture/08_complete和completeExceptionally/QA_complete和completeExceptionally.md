
# complete 和 completeExceptionally

## Q1：complete() 方法的返回值代表什么？

**A**：返回 `boolean`：
- `true`：成功设置了结果（future 之前**未完成**）
- `false`：设置失败（future **已经完成**，无论是正常完成还是异常完成）

即 `complete()` 是**幂等安全**的——多次调用只有第一次有效。

---

## Q2：什么场景下需要手动 complete 一个 CompletableFuture？

**A**：常见场景：

1. **缓存模式**：查到缓存直接 `complete()` 返回，避免异步调用
2. **测试 Mock**：手动完成 future 模拟异步结果，控制测试时序
3. **超时降级**：超时后用默认值 `complete()` 兜底
4. **事件驱动**：监听事件到达后 `complete()` 解除阻塞

---

## Q3：complete 和 obtrudeValue 有什么区别？

**A**：

- `complete(value)`：仅在 future **未完成**时生效，已完成后返回 false
- `obtrudeValue(value)`：**强制覆盖**，即使 future 已完成也会修改结果

`obtrude*` 会破坏 CompletableFuture 的正常语义（已完成的 future 结果不应变化），**生产代码中应避免使用**。

---
```java
CompletableFuture<String> cf = new CompletableFuture<>();

// 手动完成
cf.complete("done");
System.out.println(cf.join());  // "done"

// 手动设置异常
CompletableFuture<String> cf2 = new CompletableFuture<>();
cf2.completeExceptionally(new RuntimeException("出错了"));

try { cf2.join(); }
catch (CompletionException e) {
    System.out.println(e.getCause().getMessage());  // "出错了"
}

// 实际应用：超时降级
CompletableFuture<String> remote = CompletableFuture.supplyAsync(() -> remoteCall());
CompletableFuture<String> fallback = CompletableFuture.supplyAsync(() -> "fallback");
remote.completeOnTimeout("timeout", 3, TimeUnit.SECONDS)
    .exceptionally(ex -> "fallback");
```

