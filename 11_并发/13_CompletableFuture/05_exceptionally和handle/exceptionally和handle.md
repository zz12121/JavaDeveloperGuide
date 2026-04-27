
# exceptionally和handle

## 核心结论

CompletableFuture 提供 **两种异常处理机制**：

| 方法 | 触发条件 | 返回值 | 典型用途 |
|------|---------|--------|---------|
| `exceptionally(fn)` | 仅异常时触发 | 恢复值 | 异常兜底，返回默认值 |
| `handle(fn)` | 成功或异常都触发 | 转换后的值 | 统一处理，类似 try-finally |
| `whenComplete(fn)` | 成功或异常都触发 | 无返回值 | 日志记录、资源清理 |

## 深度解析

### exceptionally — 异常恢复

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (true) throw new RuntimeException("失败");
        return "OK";
    })
    .exceptionally(ex -> {
        // 仅在前置阶段异常时执行
        log.error("异常", ex);
        return "默认值"; // 返回恢复值，后续链正常继续
    });
// 结果: "默认值"
```

**关键点**：
- `exceptionally` 的函数签名是 `Function<Throwable, T>`
- 异常**被消费**，后续链不会收到异常
- 类似 try-catch 的作用

### handle — 统一处理

```java
CompletableFuture<Integer> future = CompletableFuture
    .supplyAsync(() -> "123")
    .handle((result, ex) -> {
        // result 和 ex 互斥：成功时 result 有值 ex 为 null，异常时反之
        if (ex != null) {
            log.error("出错了", ex);
            return -1;
        }
        return Integer.parseInt(result);
    });
// 结果: 123
```

### whenComplete vs handle

```java
// whenComplete: 无返回值，不改变结果
future.whenComplete((result, ex) -> {
    if (ex != null) log.error("异常", ex);
    else log.info("成功: {}", result);
});

// handle: 有返回值，可以转换结果
future.handle((result, ex) -> {
    return ex != null ? "fallback" : result;
});
```

**区别**：
- `whenComplete` 如果抛异常，会覆盖原结果
- `handle` 的返回值会替代原结果

### 异常传播链

```java
CompletableFuture<String> f = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("A"); })
    .thenApply(s -> s + " processed")  // 跳过
    .thenApply(s -> s + " again")       // 跳过
    .exceptionally(ex -> "recovered: " + ex.getCause().getMessage());
// 结果: "recovered: A"
```

异常会**跳过后续的 thenApply/thenAccept**，直到遇到 `exceptionally` 或 `handle`。

### exceptionallyAfter / handleAfter（JDK12+）

```java
// exceptionally在当前阶段执行
// exceptionallyAfter在另一个CF完成后执行异常恢复
future.exceptionallyAfter(otherFuture, ex -> fallback);
```

## 代码示例

### 完整异常处理模板

```java
public CompletableFuture<Order> getOrder(String orderId) {
    return orderService.getOrderAsync(orderId)
        .thenCompose(order -> productService.getProductAsync(order.getProductId())
            .thenAccept(order::setProduct)
            .thenApply(v -> order))
        .handle((order, ex) -> {
            if (ex != null) {
                log.warn("获取订单{}失败，返回缓存", orderId);
                return orderCache.get(orderId); // 降级
            }
            return order;
        })
        .whenComplete((order, ex) -> {
            // 无论成功失败都记录监控指标
            metrics.record("getOrder", ex == null ? "success" : "error");
        });
}
```

### 多阶段异常捕获

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> parse(data))
    .thenApply(parsed -> transform(parsed))
    .exceptionally(ex -> {
        // 捕获任一阶段的异常
        if (ex.getCause() instanceof ParseException) {
            return "解析失败默认值";
        }
        return "其他异常默认值";
    });
```

## 易错点与踩坑

### 1. exceptionally 只处理异常，不处理正常结果
```java
// exceptionally 只在异常时触发，正常时直接跳过
CompletableFuture<String> cf = supplyAsync(() -> "OK")
    .exceptionally(ex -> "fallback"); // 不执行！因为是正常的
// cf.join() = "OK"

// 如果需要同时处理正常和异常 → 用 handle
```

### 2. handle vs exceptionally 的选择
```java
// handle: 无论成功失败都执行（类似 finally + catch）
// exceptionally: 只在失败时执行（类似 catch）

// ❌ 用 exceptionally 做日志记录（成功时无法记录）
.exceptionally(ex -> {
    log.error("任务失败", ex);
    return "default";
});
// 成功时没有任何日志

// ✅ 用 handle 记录日志
.handle((result, ex) -> {
    if (ex != null) {
        log.error("任务失败", ex);
        return "default";
    }
    log.info("任务成功: {}", result);
    return result;
});
```

### 3. exceptionally 中的异常会导致二次异常
```java
// ❌ exceptionally 本身抛异常 → 新异常替换原异常
.exceptionally(ex -> {
    throw new RuntimeException("处理失败", ex); // exceptionally 中抛异常
});
// 最终 get() 得到的是新的 RuntimeException，不是原始异常

// ✅ exceptionally 中不要抛异常
.exceptionally(ex -> {
    log.error("原始异常", ex);
    return "safe-default"; // 必须返回一个正常值
});
```

### 4. exceptionally 不会恢复上游的"已完成"状态
```java
// exceptionally 返回一个新值，但原始 CF 仍然是异常状态
CompletableFuture<String> original = supplyAsync(() -> { throw new RuntimeException("fail"); });
CompletableFuture<String> recovered = original.exceptionally(ex -> "recovered");

original.isCompletedExceptionally(); // true
recovered.join(); // "recovered"
// original 和 recovered 是两个不同的 CF
```

### 5. whenComplete 抛异常会覆盖原异常（经典坑）

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        if (true) throw new RuntimeException("原始异常");
        return "OK";
    })
    .whenComplete((result, ex) -> {
        if (ex != null) {
            // ⚠️ 如果这里再抛异常，原始异常会被替换！
            throw new RuntimeException("whenComplete中又出错了");
        }
    });

try {
    cf.join();
} catch (CompletionException e) {
    System.out.println(e.getCause()); // RuntimeException("whenComplete中又出错了")
    // ❌ 原始的 RuntimeException("原始异常") 丢失了！
}
```

**正确做法**：whenComplete 只做日志/监控，不要抛异常

```java
// ✅ whenComplete 中只记录，不要抛异常
cf.whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("原始异常被吞掉了！原始异常:", ex); // 只记录，不抛
    }
});

// ✅ 如果必须处理异常，用 handle
cf.handle((result, ex) -> {
    if (ex != null) {
        log.error("异常", ex);
        return "fallback"; // 返回降级值
    }
    return result;
});
```

### 6. handle 的 function 抛异常会替换原异常

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        throw new RuntimeException("原始异常");
    })
    .handle((result, ex) -> {
        if (ex != null) {
            // ⚠️ function 抛异常 → 替换原异常
            throw new IllegalStateException("handle内部错误");
        }
        return result;
    });

try {
    cf.join();
} catch (CompletionException e) {
    System.out.println(e.getCause()); // IllegalStateException("handle内部错误")
    // ❌ 原始的 RuntimeException("原始异常") 丢失了！
}
```

**正确做法**：function 中要抛异常时，先处理再抛

```java
// ✅ 方案1：function 中不抛异常
.handle((result, ex) -> {
    if (ex != null) {
        log.error("处理异常", ex);
        return "fallback"; // 正常返回降级值
    }
    return result;
});

// ✅ 方案2：需要抛异常时用 exceptionallyAfter
.handle((result, ex) -> {
    if (ex != null) {
        throw CompletionException.of(ex); // 保留原始异常
    }
    return result;
});
```

### 7. exceptionally vs handle vs whenComplete 异常行为对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        异常处理器行为对比                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  方法            │ 函数抛异常      │ 函数返回值                          │
│  ───────────────┼────────────────┼────────────────────────────────    │
│  exceptionally  │ 替换原异常      │ 替换为返回值（后续链正常）           │
│  handle         │ 替换原异常      │ 替换为返回值（后续链正常）           │
│  whenComplete   │ 替换原异常 ⚠️   │ 无返回值（结果不变）                 │
│  thenCompose    │ 替换原异常      │ 作为新 CF 的结果继续                │
├─────────────────────────────────────────────────────────────────────────┤
│  共同点：函数抛异常都会替换原始异常！                                      │
│  建议：这些方法中只做日志/监控，不要抛异常                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8. thenApply/thenAccept/thenRun 中的异常

```java
// thenApply 抛异常 → 替换原异常（但原异常是成功，所以直接是新的）
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> "OK")
    .thenApply(s -> {
        throw new RuntimeException("thenApply异常");
    });

cf.join(); // 抛 CompletionException(RuntimeException("thenApply异常"))

// thenCompose 抛异常 → 替换原异常
CompletableFuture<String> cf2 = CompletableFuture
    .supplyAsync(() -> "OK")
    .thenCompose(s -> {
        throw new RuntimeException("thenCompose异常");
    });

cf2.join(); // 抛 CompletionException(RuntimeException("thenCompose异常"))
```
## 关联知识点

