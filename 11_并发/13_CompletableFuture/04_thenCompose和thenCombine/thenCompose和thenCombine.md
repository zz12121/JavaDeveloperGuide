
# thenCompose

## 核心结论

`thenCompose` 用于**扁平化组合两个有依赖的异步任务**，类似 Stream 的 `flatMap`。当前一个任务返回 `CompletableFuture<CompletableFuture<T>>` 时，用 thenCompose 将嵌套展平为 `CompletableFuture<T>`，避免双层嵌套。

## 深度解析

### thenApply vs thenCompose

```java
// thenApply：返回值直接包装，产生嵌套
CompletableFuture<CompletableFuture<String>> nested =
    supplyAsync(() -> getUserId())
    .thenApply(id -> getUserAsync(id));  // 返回 CF<CF<String>>

// thenCompose：自动展平，不嵌套
CompletableFuture<String> flat =
    supplyAsync(() -> getUserId())
    .thenCompose(id -> getUserAsync(id));  // 返回 CF<String>
```

| 对比 | thenApply | thenCompose |
|------|----------|-------------|
| 函数签名 | T → R | T → CompletableFuture\<R\> |
| 嵌套问题 | CF\<CF\<R\>\> | CF\<R\>（自动展平） |
| 类比 | map | flatMap |
| 适用场景 | 同步转换 | 异步链式依赖 |

### 为什么需要展平？

```java
// 假设两个异步方法
CompletableFuture<User> getUserAsync(String id) { ... }
CompletableFuture<Order> getOrderAsync(User user) { ... }

// ❌ thenApply 嵌套
CompletableFuture<CompletableFuture<Order>> bad =
    getUserAsync("123")
    .thenApply(user -> getOrderAsync(user));  // CF<CF<Order>>
// 获取结果需要 result.get().get() — 两层 get！

// ✅ thenCompose 扁平化
CompletableFuture<Order> good =
    getUserAsync("123")
    .thenCompose(user -> getOrderAsync(user));  // CF<Order>
// 获取结果只需 result.get() — 一层 get！
```

### 实战：多步异步依赖

```java
// 用户登录 → 获取权限 → 加载页面
CompletableFuture<Page> loadPage = CompletableFuture
    .supplyAsync(() -> login(username, password))     // CF<User>
    .thenCompose(user -> getPermissions(user))        // CF<List<String>>
    .thenCompose(perms -> loadDashboard(perms));      // CF<Page>
```

## 易错点与踩坑

### 1. thenCompose vs thenApply 返回类型混淆（高频问题）
```java
// ❌ 错误：thenApply 返回嵌套的 CompletableFuture
CompletableFuture<CompletableFuture<String>> wrong = 
    CompletableFuture.supplyAsync(() -> "id")
    .thenApply(id -> CompletableFuture.supplyAsync(() -> queryUser(id)));
// 结果类型: CompletableFuture<CompletableFuture<String>>

// ✅ 正确：thenCompose 自动扁平化
CompletableFuture<String> correct = 
    CompletableFuture.supplyAsync(() -> "id")
    .thenCompose(id -> CompletableFuture.supplyAsync(() -> queryUser(id)));
// 结果类型: CompletableFuture<String>

// 类比 Stream: thenApply = map, thenCompose = flatMap
```

### 2. thenCompose 中返回已完成的 CF
```java
// 如果第二个操作不需要异步 → 返回 completedFuture
.thenCompose(id -> {
    if (cache.containsKey(id)) {
        return CompletableFuture.completedFuture(cache.get(id)); // 缓存命中
    }
    return CompletableFuture.supplyAsync(() -> db.query(id), ioPool); // 缓存未命中
});
```

### 3. thenCompose 中的异常处理
```java
// thenCompose 中的第一个 CF 异常 → 整条链异常
// 第二个 CF 异常 → 也导致整条链异常
supplyAsync(() -> "id")
    .thenCompose(id -> supplyAsync(() -> {
        if (id == null) throw new IllegalArgumentException("id不能为空");
        return queryUser(id);
    }))
    .exceptionally(ex -> {
        // 第一个或第二个 CF 异常都会到这里
        log.error("查询失败", ex);
        return null;
    });
```

### 4. 链式 thenCompose 的延迟执行
```java
// thenCompose 中的 Supplier 是惰性的：只有前一步完成后才执行
// 不会提前执行
CompletableFuture<String> step1 = supplyAsync(() -> "A");
CompletableFuture<String> step2 = step1.thenCompose(a -> 
    supplyAsync(() -> a + "B")  // step1 完成后才提交
);
// step2 的异步任务此时还未提交到线程池
```


# thenCombine

## 核心结论

`thenCombine` 用于**组合两个独立的异步任务**，当两个任务都完成时，将它们的结果合并为一个新结果。与 thenCompose 不同（依赖链式），thenCombine 的两个任务是**并行执行**的，最后通过 BiFunction 合并结果。

## 深度解析

### thenCompose vs thenCombine

| 对比 | thenCompose | thenCombine |
|------|------------|-------------|
| 任务关系 | 有依赖（串行） | 独立（并行） |
| 执行方式 | A 完成 → 执行 B | A 和 B 同时执行 |
| 合并时机 | B 的结果依赖 A | A 和 B 都完成后合并 |
| 类比 | flatMap | zip |

### 基本用法

```java
CompletableFuture<String> futureA = supplyAsync(() -> fetchUserName());
CompletableFuture<Integer> futureB = supplyAsync(() -> fetchUserAge());

CompletableFuture<String> combined = futureA.thenCombine(futureB, (name, age) -> {
    return name + " is " + age + " years old";
});
```

### 执行时序

```
thenCombine：
┌─ futureA (并行) ──→ 结果A ─┐
                              ├─→ BiFunction(A, B) → 合并结果
┌─ futureB (并行) ──→ 结果B ─┘

thenCompose：
┌─ futureA ──→ 结果A ──→ futureB(依赖A) ──→ 最终结果
```

### 实战：并行查询合并

```java
CompletableFuture<User> userFuture = supplyAsync(() -> userMapper.selectById(userId), dbPool);
CompletableFuture<List<Order>> orderFuture = supplyAsync(() -> orderMapper.selectByUserId(userId), dbPool);

CompletableFuture<UserVO> voFuture = userFuture.thenCombine(orderFuture, (user, orders) -> {
    UserVO vo = new UserVO();
    vo.setName(user.getName());
    vo.setOrders(orders);
    return vo;
});
```

### 相关方法

| 方法 | 说明 |
|------|------|
| `thenCombine(other, fn)` | 合并结果，有返回值 |
| `thenAcceptBoth(other, consumer)` | 合并结果，无返回值 |
| `allOf(cfs...)` | 等待所有 CF 完成 |
| `anyOf(cfs...)` | 等待任一 CF 完成 |

## 易错点与踩坑

### 1. 两个任务中一个异常导致整个 thenCombine 失败
```java
// ❌ futureB 异常 → thenCombine 不执行合并函数
CompletableFuture<String> futureA = supplyAsync(() -> "hello");
CompletableFuture<Integer> futureB = supplyAsync(() -> {
    throw new RuntimeException("B 失败");
});
futureA.thenCombine(futureB, (a, b) -> a + b); // 永远不会执行合并

// ✅ 对每个 future 分别做异常处理
CompletableFuture<Integer> safeB = futureB.exceptionally(ex -> 0);
futureA.thenCombine(safeB, (a, b) -> a + b); // "hello0"
```

### 2. thenCombine vs thenAcceptBoth 的选择
```java
// thenCombine → 有返回值
// thenAcceptBoth → 无返回值（终点操作）

// ❌ 不需要返回值却用 thenCombine（浪费）
futureA.thenCombine(futureB, (a, b) -> {
    log.info("{} + {}", a, b);
    return null; // 没人用这个返回值
});

// ✅ 用 thenAcceptBoth
futureA.thenAcceptBoth(futureB, (a, b) -> {
    log.info("{} + {}", a, b);
});
```

### 3. thenCombineAsync 的线程陷阱
```java
// thenCombine 的合并函数在哪个线程执行？
// 如果 futureA 先完成 → 合并函数在 futureA 完成的线程中执行
// 如果 futureB 先完成 → 合并函数在 futureB 完成的线程中执行
// 不是"两个都完成后才在新线程执行"

// 如果合并函数是重量级操作 → 用 thenCombineAsync 指定线程池
futureA.thenCombineAsync(futureB, mergeFn, cpuPool);
```

### 4. 三个以上任务的组合
```java
// thenCombine 只能组合两个 → 三个及以上嵌套很深
// ✅ 用 allOf 等待所有完成，再手动获取结果
CompletableFuture<String> a = supplyAsync(() -> fetchA());
CompletableFuture<String> b = supplyAsync(() -> fetchB());
CompletableFuture<String> c = supplyAsync(() -> fetchC());

CompletableFuture.allOf(a, b, c).thenRun(() -> {
    String ra = a.join();
    String rb = b.join();
    String rc = c.join();
    // 合并处理
});
```
## 关联知识点
