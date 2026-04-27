# obtrude 危险方法专题

## 核心结论

`obtrudeValue()` 和 `obtrudeException()` 是 CompletableFuture 的**危险方法**：

- 绕过"只完成一次"的语义
- 强制覆盖已有状态
- **生产代码禁止使用**

```
complete()    →  只在未完成时设置 ✅
obtrudeValue()→  无论是否完成都强制设置 ⚠️
```

## 方法签名

```java
// JDK 9+ 已标记为 @Deprecated
@Deprecated
public void obtrudeValue(T value) { ... }

@Deprecated
public void obtrudeException(Throwable ex) { ... }
```

## 深度解析

### obtrudeValue — 强制设置正常值

```java
CompletableFuture<String> cf = new CompletableFuture<>();

// 正常流程
cf.complete("A");
cf.join(); // "A"

// obtrudeValue：强制覆盖，即使已经完成
CompletableFuture<String> cf2 = CompletableFuture.completedFuture("原始值");
cf2.obtrudeValue("被强制覆盖了");
cf2.join(); // "被强制覆盖了"
```

### obtrudeException — 强制设置异常

```java
// 正常完成的 CF，强制设置为异常
CompletableFuture<String> cf = CompletableFuture.completedFuture("OK");
cf.obtrudeException(new RuntimeException("被强制异常了"));
cf.join(); // 抛 CompletionException(RuntimeException("被强制异常了"))
```

### 危险场景分析

#### 危险1：破坏已有结果

```java
// 场景：缓存场景
CompletableFuture<String> cache = new CompletableFuture<>();

// 线程A：查询到数据，完成 CF
cache.complete("cached data");

// 线程B：查询数据库失败，试图强制覆盖
// ⚠️ 如果此时用 obtrude，会丢失已缓存的数据
cache.obtrudeException(new RuntimeException("DB error"));

// 结果：缓存数据丢失
```

#### 危险2：异常传播链断裂

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("原始异常"); })
    .thenApply(s -> s + " processed")
    .exceptionally(ex -> "recovered");

// 外部强制覆盖结果，破坏异常传播
cf.obtrudeValue("forced value");

// ❌ 原始异常丢失
cf.exceptionally(ex -> {
    // 这个不会执行，因为已经被强制完成了
    log.error("永远执行不到");
    return "fallback";
}).join();

// ✅ obtrude 之后的 exceptionally 仍然有效
cf.exceptionally(ex -> {
    log.error("obtrude后异常");
    return "fallback";
}).join(); // "forced value"（因为 CF 已完成且正常，exceptionally 不执行）
```

#### 危险3：状态不一致

```java
// obtrudeValue 会使 isDone() 为 true，但 isCompletedExceptionally() 仍可能为 true
CompletableFuture<String> cf = new CompletableFuture<>();
cf.completeExceptionally(new RuntimeException("fail"));
cf.obtrudeValue("forced"); // 强制覆盖为正常

// 状态分析
cf.isDone();                    // true
cf.isCompletedExceptionally();   // true（⚠️ 状态不一致！）
cf.join();                      // "forced"（正常返回）
```

### 为什么 JDK 标记为 @Deprecated

1. **违反契约**：`Future` 接口约定只在未完成时设置值
2. **难以调试**：强制修改后，异常传播链断裂，难以追踪问题
3. **线程不安全**：多个线程同时 obtrude 可能导致不可预期行为
4. **生产危险**：容易引入隐蔽 bug

## 正确替代方案

### 场景1：需要覆盖已有结果

```java
// ❌ 不要用 obtrude
cf.obtrudeValue("new value");

// ✅ 用新的 CF
CompletableFuture<String> newCf = CompletableFuture.completedFuture("new value");

// ✅ 或用 complete + 状态检查
if (!cf.isDone()) {
    cf.complete("new value");
} else {
    // 已完成，考虑其他策略
}
```

### 场景2：需要强制标记为异常

```java
// ❌ 不要用 obtrude
cf.obtrudeException(new RuntimeException("fail"));

// ✅ 用 completeExceptionally（但要确保之前未完成）
if (!cf.isDone()) {
    cf.completeExceptionally(new RuntimeException("fail"));
}
```

### 场景3：超时后降级

```java
// ❌ 不要用 obtrude
CompletableFuture<String> cf = httpClient.getAsync();
scheduled.schedule(() -> {
    cf.obtrudeValue("timeout fallback"); // 危险
}, 3, TimeUnit.SECONDS);

// ✅ 用 complete + 状态检查
CompletableFuture<String> cf = httpClient.getAsync();
scheduled.schedule(() -> {
    if (!cf.isDone()) {
        cf.complete("timeout fallback"); // 安全
    }
}, 3, TimeUnit.SECONDS);
```

## 易错点与踩坑

### 1. 误用 obtrude 导致数据丢失

```java
// ❌ 典型错误
CompletableFuture<String> cf = new CompletableFuture<>();
saveToCache(cf);
someAsyncOperation().whenComplete((r, ex) -> {
    if (ex != null) {
        cf.obtrudeException(ex); // ⚠️ 危险！
    }
});

// ✅ 正确做法
CompletableFuture<String> cf = new CompletableFuture<>();
saveToCache(cf);
someAsyncOperation().whenComplete((r, ex) -> {
    if (ex != null && !cf.isDone()) {
        cf.completeExceptionally(ex);
    }
});
```

### 2. 混淆 complete 和 obtrudeValue

```java
// complete：只在未完成时设置
cf.complete("A");    // true
cf.complete("B");    // false，已完成

// obtrudeValue：无论是否完成都设置
cf.obtrudeValue("C"); // 无返回值，始终覆盖
```

### 3. obtrude 后异常处理器不生效

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("原始"); })
    .exceptionally(ex -> "recovered");

// obtrude 强制覆盖为正常值
cf.obtrudeValue("forced");

// exceptionallly 不会执行，因为 CF 已正常完成
cf.exceptionally(ex -> {
    log.error("不会执行");
    return "fallback";
}).join(); // "forced"
```

## 总结

```
┌──────────────────────────────────────────────────────────────┐
│                    obtrude 方法使用指南                         │
├──────────────────────────────────────────────────────────────┤
│  ✅ use when：测试代码中模拟异常                               │
│  ❌ don't use：生产代码                                        │
├──────────────────────────────────────────────────────────────┤
│  obtrudeValue()     →  强制设置正常值                          │
│  obtrudeException() →  强制设置异常                           │
├──────────────────────────────────────────────────────────────┤
│  ⚠️ 危险：                                                    │
│  1. 绕过"只完成一次"语义                                       │
│  2. 破坏异常传播链                                             │
│  3. 导致状态不一致                                            │
│  4. JDK 已标记 @Deprecated                                     │
├──────────────────────────────────────────────────────────────┤
│  ✅ 替代方案：                                                 │
│  if (!cf.isDone()) cf.complete(value);                        │
│  if (!cf.isDone()) cf.completeExceptionally(ex);              │
└──────────────────────────────────────────────────────────────┘
```

## 关联知识点

- [[complete和completeExceptionally|complete 与 completeExceptionally]]
- [[Cancellation专题|Cancellation 机制]]
