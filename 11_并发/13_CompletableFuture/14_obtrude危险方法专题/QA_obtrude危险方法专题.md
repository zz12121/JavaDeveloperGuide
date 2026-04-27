# obtrude危险方法专题

## Q1：obtrudeValue 和 complete 有什么区别？

**A**：

| 方法 | 未完成时 | 已完成时 | 返回值 |
|------|---------|---------|--------|
| `complete(value)` | 设置为 value | 返回 false，不设置 | boolean |
| `obtrudeValue(value)` | 设置为 value | 强制覆盖为 value | void |

```java
CompletableFuture<String> cf = new CompletableFuture<>();

// 初始状态
cf.complete("A");         // true，cf = "A"
cf.complete("B");         // false，cf 仍是 "A"

// obtrudeValue 无视完成状态
cf.obtrudeValue("C");    // 强制覆盖，cf = "C"
```

---

## Q2：为什么 obtrude 方法被标记为 @Deprecated？

**A**：四个原因：

1. **违反契约**：`Future` 接口约定只在未完成时设置值
2. **难以调试**：强制覆盖后，异常传播链断裂
3. **线程不安全**：多线程同时 obtrude 不可预期
4. **生产危险**：引入隐蔽 bug

JDK 9+ 已标记，建议生产环境禁用。

---

## Q3：obtrudeException 和 completeExceptionally 有什么区别？

**A**：

| 方法 | 行为 |
|------|------|
| `completeExceptionally(ex)` | 只在未完成时设置异常，已完成时无效 |
| `obtrudeException(ex)` | 无论是否完成都强制设置异常 |

```java
// 场景：CF 已正常完成
CompletableFuture<String> cf = CompletableFuture.completedFuture("OK");

cf.completeExceptionally(new RuntimeException("fail"));
// 无效！cf 仍是 "OK"

cf.obtrudeException(new RuntimeException("forced"));
// 强制覆盖为异常，cf.join() 抛 CompletionException
```

---

## Q4：obtrude 后 exceptionally 还会执行吗？

**A**：取决于 obtrude 的值：

```java
// obtrudeValue：CF 已正常完成，exceptionally 不执行
cf.obtrudeValue("forced");
cf.exceptionally(ex -> "fallback").join(); // "forced"

// obtrudeException：CF 已异常完成，exceptionally 执行
cf.obtrudeException(new RuntimeException("forced"));
cf.exceptionally(ex -> "recovered").join(); // "recovered"
```

---

## Q5：生产代码中如何替代 obtrude？

**A**：用 complete/completeExceptionally + 状态检查：

```java
// ❌ 不要用
cf.obtrudeValue("fallback");

// ✅ 正确做法
if (!cf.isDone()) {
    cf.complete("fallback");
}

// ✅ 或者用 handle 做统一处理
cf.handle((result, ex) -> {
    if (ex != null) {
        return "fallback";
    }
    return result;
});
```
