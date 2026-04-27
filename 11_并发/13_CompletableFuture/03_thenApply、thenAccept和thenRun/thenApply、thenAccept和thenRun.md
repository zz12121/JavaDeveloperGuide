
# thenApply/thenAccept/thenRun

## 核心结论

这三个方法是 CompletableFuture 链式调用的核心，区别在于**是否接收参数**和**是否返回值**：
- `thenApply(fn)` — 接收结果 + 返回新值（转换）
- `thenAccept(fn)` — 接收结果 + 无返回值（消费）
- `thenRun(fn)` — 不接收结果 + 无返回值（执行）

## 深度解析

### 对比

| 方法 | 参数 | 返回值 | 类比 |
|------|------|--------|------|
| `thenApply(T → R)` | 接收 T | 返回 CompletableFuture\<R\> | Stream.map |
| `thenAccept(T → void)` | 接收 T | 返回 CompletableFuture\<Void\> | Stream.forEach |
| `thenRun(() → void)` | 无参数 | 返回 CompletableFuture\<Void\> | finally 块 |

### 链式调用示例

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchUserId())          // → String (userId)
    .thenApply(id -> queryUserById(id))        // → User
    .thenApply(user -> user.getName())         // → String (name)
    .thenAccept(name -> System.out.println(name)) // → Void (打印)
    .thenRun(() -> log.info("Done"));           // → Void (记录日志)
```

### 具体用法

```java
// thenApply：转换结果，有返回值
CompletableFuture<Integer> f1 = CompletableFuture
    .supplyAsync(() -> "100")
    .thenApply(Integer::parseInt)       // String → Integer
    .thenApply(n -> n * 2);             // Integer → Integer (200)

// thenAccept：消费结果，无返回值
CompletableFuture<Void> f2 = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenAccept(s -> System.out.println(s));  // 打印，返回 Void

// thenRun：不关心上一步结果，执行副作用
CompletableFuture<Void> f3 = CompletableFuture
    .supplyAsync(() -> fetchData())
    .thenRun(() -> System.out.println("Data fetched"));  // 不用上一步的数据
```

### 选择指南

```
需要前一步的结果吗？
├── 不需要 → thenRun
└── 需要
    └── 需要产生新结果给下一步吗？
        ├── 需要 → thenApply
        └── 不需要（终点操作）→ thenAccept
```

## 易错点与踩坑

### 1. thenApply 中抛异常导致后续链断裂
```java
// ❌ thenApply 抛异常 → 后续的 thenAccept/thenRun 不执行
CompletableFuture.supplyAsync(() -> "100")
    .thenApply(s -> Integer.parseInt(s))    // 正常
    .thenApply(n -> 100 / n)               // 如果 n=0 → ArithmeticException
    .thenAccept(result -> System.out.println(result));  // 不执行！整个链进入异常状态

// ✅ 用 exceptionally 或 handle 兜底
.thenApply(n -> 100 / n)
.exceptionally(ex -> {
    log.error("计算异常", ex);
    return 0; // 降级值
})
.thenAccept(result -> System.out.println(result)); // 正常执行
```

### 2. thenRun 不接收参数但不代表没有副作用
```java
// thenRun 的常见误解：以为整个链没用了
// 实际上 thenRun 常用于"收尾工作"
supplyAsync(() -> fetchData())
    .thenAccept(data -> processData(data))
    .thenRun(() -> log.info("处理完成，发送通知"))
    .thenRun(() -> metrics.increment("processed"));
// 每个 thenRun 都会执行（除非前面的步骤异常）
```

### 3. 链式调用返回值丢失
```java
// ❌ thenAccept 后返回 CompletableFuture<Void>，后续无法获取结果
CompletableFuture<Void> cf = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenAccept(s -> System.out.println(s)); // 返回 Void
// cf.join() 返回 null

// ✅ 需要结果时用 thenApply 而非 thenAccept
CompletableFuture<String> cf2 = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenApply(s -> {
        System.out.println(s);
        return s; // 返回结果
    });
```

### 4. thenAccept 的 NullPointerException
```java
// supplyAsync 返回 null → thenAccept 接收 null
CompletableFuture.supplyAsync(() -> null)
    .thenAccept(s -> s.length());  // s 为 null → NPE

// ✅ 增加空判断
.thenAccept(s -> {
    if (s != null) s.length();
});
```

### 5. thenApply 接收上游 null 的处理

```java
// ⚠️ 上游返回 null → thenApply 接收 null
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> findUser())  // 可能返回 null
    .thenApply(user -> user.getName()); // user 为 null → NPE

// ✅ 方案1：增加 null 检查
.thenApply(user -> {
    if (user == null) return "anonymous";
    return user.getName();
});

// ✅ 方案2：用 Optional 包装
.thenApply(user -> Optional.ofNullable(user)
    .map(User::getName)
    .orElse("anonymous"));

// ✅ 方案3：用 handle 统一处理
.handle((user, ex) -> {
    if (ex != null || user == null) {
        return "anonymous";
    }
    return user.getName();
});
```

### 6. thenApply 返回 null 的影响

```java
// ⚠️ thenApply 返回 null → 后续接收 null
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenApply(s -> null); // 返回 null

cf.join(); // null（不是异常）

// ⚠️ 后续 thenAccept 接收 null
cf.thenAccept(s -> s.length()); // NPE！

// ✅ 返回值用 Optional 表示"可能为空"
.thenApply(s -> Optional.ofNullable(s));
```

### 7. null 在异步链中的传播

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      null 在异步链中的传播                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  supplyAsync(() -> null)                                                  │
│         │                                                                 │
│         ▼                                                                 │
│  thenApply(fn) ── 接收 null ──► 返回值可能是 null                          │
│         │                                                                 │
│         ▼                                                                 │
│  thenAccept(fn) ── 接收 null ──► NPE！                                     │
│                                                                            │
│  ⚠️ 关键点：                                                              │
│  - null 不是异常，不会触发 exceptionally/handle 的异常分支                    │
│  - null 会正常传播，直到被某个操作消费（NPE）                                  │
│  - 建议：使用 Optional<T> 代替 null                                        │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 8. null 安全实践

```java
// 模式1：用 Optional 包装输入输出
CompletableFuture<Optional<User>> cf = CompletableFuture
    .supplyAsync(() -> findUser(id))
    .thenApply(Optional::ofNullable)  // null → Optional.empty()

    .thenApply(optUser -> optUser
        .map(User::getName)
        .orElse("anonymous"));

// 模式2：用 null 对象模式
.thenApply(user -> user != null ? user : User.NULL);

// 模式3：用 handle 统一处理
.handle((result, ex) -> {
    if (ex != null || result == null) {
        return getDefault();
    }
    return result;
});
```
## 关联知识点