
# StructuredTaskScope

## Q1：什么是结构化并发？StructuredTaskScope 解决了什么问题？

**A**：

结构化并发（Structured Concurrency）是一种并发编程范式，核心思想是：**将并发子任务组织成有明确生命周期的任务树，父任务必须等待所有子任务完成或终止**。

StructuredTaskScope 解决了传统并发编程的三大痛点：

| 痛点 | 传统方式（线程池/Future） | StructuredTaskScope |
|------|------------------------|-------------------|
| **任务泄漏** | 提交后难以追踪，容易忘记等待 | try-with-resources 强制等待所有子任务 |
| **异常丢失** | 异常被吞或传播不清晰 | `throwIfFailed()` 统一异常处理 |
| **取消困难** | 需要手动逐个 cancel | 策略自动取消（失败/成功时） |

类比：结构化并发之于线程，就像结构化编程（if/for/while）之于 goto —— 让控制流清晰可追踪。

## Q2：StructuredTaskScope 有哪些策略？各自适用场景？

**A**：

两种内置策略：

**1. ShutdownOnFailure（AND 语义）**
- **行为**：任一子任务失败 → 立即取消所有其他子任务
- **适用场景**：需要聚合多个依赖结果的场景
- **示例**：同时查询用户信息和订单信息，任一失败则整体失败

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> fetchUser(id));
    scope.fork(() -> fetchOrder(id));
    scope.join();
    scope.throwIfFailed(); // 任一失败 → 其余已取消，直接抛异常
}
```

**2. ShutdownOnSuccess（OR 语义）**
- **行为**：首个子任务成功 → 立即取消所有其他子任务
- **适用场景**：多源竞争/容错查询
- **示例**：从多个缓存/数据源竞争查询，谁先返回用谁

```java
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> fetchFromRedis(key));
    scope.fork(() -> fetchFromDB(key));
    scope.join();
    return scope.result(); // 第一个成功的结果
}
```

## Q3：StructuredTaskScope 和 CompletableFuture 有什么区别？

**A**：

| 维度 | StructuredTaskScope | CompletableFuture |
|------|-------------------|-------------------|
| **编程模型** | 同步阻塞（类似普通方法调用） | 异步链式回调 |
| **任务管理** | 结构化，父子关系明确 | 扁平化，手动管理 |
| **异常传播** | `throwIfFailed()` 直接抛出 | `exceptionally()` 链式处理 |
| **取消机制** | 策略自动取消 | 手动 `cancel()` |
| **可观测性** | jstack 可见任务父子关系 | 无法追踪任务来源 |
| **适用场景** | 多个子任务并发执行后聚合 | 异步流水线、事件驱动编排 |

**选择建议**：
- 需要并发执行多个独立任务并聚合结果 → **StructuredTaskScope**
- 需要复杂的异步流水线编排 → **CompletableFuture**
- 两者可以结合使用：在 StructuredTaskScope 中 fork 的任务内部使用 CompletableFuture 编排细节

## Q4：StructuredTaskScope 在哪个 JDK 版本正式发布的？

**A**：

- **JDK 19（2022）**：首次预览，位于 `jdk.incubator.concurrent` 模块
- **JDK 21（2023）**：第二次预览（JEP 453）
- **JDK 23（2024）**：**正式发布**，移至 `java.util.concurrent` 包，无需 `--enable-preview`

> 实际使用中 JDK 21 预览版也可用，但需要编译和运行时都加 `--enable-preview --add-modules jdk.incubator.concurrent`。推荐在 JDK 23+ 直接使用正式版。

---

## Q5：joinUntil() 是什么？和 join() 有什么区别？

**A**：

`joinUntil(Instant deadline)` 是 JDK 23 新增的方法，与 `join()` 的区别在于**支持超时截止**：

```java
// join()：无限等待，直到所有子任务完成
scope.join();

// joinUntil()：到截止时间就停止等待
Instant deadline = Instant.now().plusSeconds(5);
try {
    scope.joinUntil(deadline);
} catch (TimeoutException e) {
    // 超时后，父线程不再等待，但子任务仍在运行
    scope.close();  // ⚠️ 必须手动关闭，否则子任务泄漏
    throw new ServiceException("超时");
}
```

**使用场景**：

```java
// 查询多个服务，5秒内必须返回结果
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Instant deadline = Instant.now().plusSeconds(5);
    scope.fork(() -> fetchFromRedis(key));  // 虚拟线程
    scope.fork(() -> fetchFromDB(key));      // 虚拟线程
    scope.fork(() -> fetchFromCache(key));   // 虚拟线程

    try {
        scope.joinUntil(deadline);
    } catch (TimeoutException e) {
        logger.warn("查询超时，取消未完成任务");
        scope.close();  // 关闭 scope，取消所有子任务
        throw new ServiceException("超时");
    }

    scope.throwIfFailed();
    var result = ...
}
```

**超时后的行为**：

| 行为 | 说明 |
|------|------|
| 子任务状态 | 继续运行（不在 scope 管控内） |
| 建议操作 | 必须调用 `scope.close()` 取消子任务 |
| scope 状态 | 仍处于打开状态，不自动关闭 |

---

## Q6：StructuredTaskScope 忘记 close() 会有什么后果？

**A**：

**后果：子任务泄漏。** 所有 fork 出来的子任务继续在后台运行，不受控制。

```java
// ❌ 错误：忘记 close()，子任务泄漏
var scope = new StructuredTaskScope.ShutdownOnFailure();
scope.fork(() -> fetchData());  // 任务在后台永远运行
scope.fork(() -> sendNotification());  // 任务在后台永远运行
scope.join();
// scope 没有 close() → 子任务泄漏，资源不释放

// ✅ 正确：使用 try-with-resources
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> fetchData());
    scope.fork(() -> sendNotification());
    scope.join();
    scope.throwIfFailed();
}
// 离开 try 块自动 close()，子任务被取消或等待完成
```

**为什么不会自动 close()？**

`StructuredTaskScope` 设计为**显式作用域管理**，让你明确知道何时结束所有子任务。如果自动关闭，你就无法控制子任务的取消时机。

**实际开发中的常见错误**：

```java
// ❌ 嵌套 scope：子 scope 在父 scope 外创建
var scope = new StructuredTaskScope.ShutdownOnFailure<>();
try {
    scope.fork(() -> {
        // ❌ 不要在 fork 内创建新的 scope
        var innerScope = new StructuredTaskScope.ShutdownOnFailure<>();
        innerScope.fork(() -> task());
        innerScope.join();  // 逻辑混乱，子任务管理困难
    });
} finally {
    scope.close();
}

// ✅ 正确：保持 scope 扁平，所有 fork 在同一 scope 内
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> task1());
    scope.fork(() -> task2());
    scope.join();
}
```

