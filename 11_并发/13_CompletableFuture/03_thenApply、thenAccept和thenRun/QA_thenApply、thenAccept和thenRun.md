
# thenApply/thenAccept/thenRun

## Q1：thenApply、thenAccept、thenRun 三者有什么区别？

**A**：
- **thenApply(T → R)**：接收上一个阶段的结果 T，返回新值 R，类型变化（类似 Stream.map）
- **thenAccept(T → void)**：接收上一个阶段的结果 T，无返回值，适合消费结果（类似 Stream.forEach）
- **thenRun(() → void)**：不接收结果，无返回值，适合执行清理/日志等副作用

选择标准：需要结果吗？→ 需要新结果就用 thenApply；只需消费就用 thenAccept；不需要上一步结果就用 thenRun。

---

## Q2：thenApply 链式调用中，类型是怎么传递的？

**A**：每个 thenApply 的返回类型由 Function 的返回类型决定，类型自动推导：

```java
CompletableFuture<String> chain = CompletableFuture
    .supplyAsync(() -> 1)           // → CompletableFuture<Integer>
    .thenApply(i -> i + 1)         // → CompletableFuture<Integer> (2)
    .thenApply(i -> "num:" + i)    // → CompletableFuture<String> ("num:2")
    .thenApply(s -> s.length());   // → CompletableFuture<Integer> (5)
```

每个阶段完成后，结果自动传递给下一个阶段作为输入。

---

## Q3：thenAccept 后面还能继续链式调用吗？

**A**：可以，但 thenAccept 返回的是 `CompletableFuture<Void>`，后续的 thenApply/thenAccept 无法接收参数（因为 Void 没有值）：

```java
supplyAsync(() -> getData())
    .thenAccept(data -> save(data))    // → CompletableFuture<Void>
    .thenRun(() -> log("saved"));      // ✅ 可以，不需要参数
    // .thenApply(data -> ...)         // ❌ 编译错误，Void 无值可传
```

所以 thenAccept 通常放在链的**终点**，前面用 thenApply 做转换。

---

## Q4：thenApply 中接收到 null 会怎样？

**A**：`null` 不会抛异常，会正常传播直到被某个操作消费：

```java
CompletableFuture.supplyAsync(() -> null)
    .thenApply(s -> s.length()); // NPE！null 被 thenApply 消费
```

**处理方式**：

```java
// ✅ 方案1：null 检查
.thenApply(s -> s != null ? s.length() : 0);

// ✅ 方案2：用 Optional
.thenApply(s -> Optional.ofNullable(s).map(String::length).orElse(0));

// ✅ 方案3：用 handle
.handle((s, ex) -> s != null ? s.length() : 0);
```

---

## Q5：thenApply 返回 null 会影响后续链吗？

**A**：`null` 会正常传递，不会抛异常：

```java
CompletableFuture.supplyAsync(() -> "hello")
    .thenApply(s -> null)        // 返回 null
    .thenApply(s -> s.toUpperCase()) // NPE！
```

**建议**：返回值也用 Optional 表示"可能为空"。

---

## Q6：null 和异常在异步链中的处理有什么不同？

**A**：

| 情况 | 是否触发 exceptionally | 是否触发 handle 的 ex | 处理方式 |
|------|----------------------|---------------------|---------|
| 上游抛异常 | ✅ | ✅（ex != null） | 异常处理器捕获 |
| 上游返回 null | ❌ | ❌（ex == null） | null 检查或 Optional |
| thenApply 抛异常 | ✅ 后续跳过后续链 | ✅（ex != null） | 异常处理器捕获 |
| thenApply 返回 null | ❌ 继续传播 | ❌ 继续传播 | null 检查 |

**核心区别**：null 不是异常，不会走异常处理路径。



