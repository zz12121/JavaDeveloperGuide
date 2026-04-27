
# StructuredTaskScope

## 核心结论

`StructuredTaskScope` 是 JDK 23 正式发布（JEP 453）的**结构化并发** API，将并发子任务组织成有明确生命周期的任务树，父任务必须等待所有子任务完成或终止。配合虚拟线程使用，能以同步代码风格写出安全、可观测、可取消的并发程序。

## 深度解析

### 什么是结构化并发

传统并发编程中，子任务以"发射后不管"的方式创建，父任务很难追踪和管理所有子任务的生命周期：

```
传统模式（非结构化）：
  父线程 → 提交 task1 到线程池（失去控制）
           → 提交 task2 到线程池（失去控制）
           → 手动 Future.get() 等待
           → 异常传播不清晰，取消困难

结构化并发模式：
  父线程 → try (scope) {
              fork task1 ─┐
              fork task2 ─┤→ scope.join() 等待所有子任务
              fork task3 ─┘  scope.throwIfFailed()
           }
           → 生命周期明确，异常自动传播，取消统一管理
```

**结构化并发的核心原则**：
- **任务生命周期有界**：子任务必须在父任务的作用域内完成
- **错误传播透明**：任一子任务失败，父任务能感知并处理
- **可观测性**：线程转储中能看到任务间的父子关系

### StructuredTaskScope 两种策略

#### ShutdownOnFailure（任一失败，全部取消）

```java
// 场景：同时查询用户信息和订单信息，任一失败则整体失败
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    StructuredTaskScope.Subtask<User> userTask =
        scope.fork(() -> fetchUser(userId));
    StructuredTaskScope.Subtask<Order> orderTask =
        scope.fork(() -> fetchOrder(orderId));

    scope.join();                // 等待所有子任务完成
    scope.throwIfFailed();       // 任一失败则抛异常，其余子任务已被取消

    // 所有任务都成功，获取结果
    User user = userTask.get();
    Order order = orderTask.get();
    return new Response(user, order);
}
// scope 自动关闭，确保所有子任务已终止
```

#### ShutdownOnSuccess（首个成功，取消其余）

```java
// 场景：从多个数据源竞争查询，谁先成功就用谁的结果
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> fetchFromSourceA(key));
    scope.fork(() -> fetchFromSourceB(key));
    scope.fork(() -> fetchFromSourceC(key));

    scope.join();
    // 第一个成功的结果，其余子任务自动取消
    return scope.result();
}
```

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **ShutdownOnFailure** | 任一子任务失败 → 取消其余，抛异常 | 聚合多个依赖的结果（AND 语义） |
| **ShutdownOnSuccess** | 首个子任务成功 → 取消其余，返回结果 | 多源竞争/容错查询（OR 语义） |

### JDK 版本历史

| 版本 | 里程碑 | JEP |
|------|--------|-----|
| JDK 19 (2022) | 首次预览（incubator） | JEP 428 |
| JDK 21 (2023) | 第二次预览 | JEP 453 |
| **JDK 23 (2024)** | **正式发布**，移至 `java.util.concurrent` | JEP 453 |

> JDK 21 的预览版需要 `--enable-preview` 和 `--add-modules jdk.incubator.concurrent`，JDK 23 正式版直接可用。

### API 核心方法

```java
// StructuredTaskScope<T> 核心方法
scope.fork(Callable<T> task)     // 创建子任务，返回 Subtask<T>
scope.join()                     // 等待所有子任务完成（响应中断）
scope.joinUntil(Instant deadline) // 等待直到截止时间，抛 TimeoutException
scope.throwIfFailed()            // 如果有子任务失败，抛出异常
scope.close()                    // 关闭作用域，等待所有子任务终止

// ShutdownOnFailure 额外方法
scope.throwIfFailed()            // 有失败则抛 ExecutionException

// ShutdownOnSuccess 额外方法
scope.result()                   // 返回第一个成功的结果
scope.result(Function<T,Exception> excMapper)  // 失败时自定义异常
```

### joinUntil() 超时控制

`joinUntil(Instant deadline)` 是 JDK 23 新增的方法，支持**截止时间式等待**，比 `join()` 更精确：

```java
// join()：无限等待
scope.join();  // 一直等到所有子任务完成

// joinUntil()：到截止时间自动抛出 TimeoutException
Instant deadline = Instant.now().plusSeconds(5);
try {
    scope.joinUntil(deadline);
} catch (TimeoutException e) {
    // 超时后 scope 内的子任务仍运行，但父线程不再等待
    scope.close();  // 建议关闭 scope 以取消子任务
    throw new TimeoutException("子任务超时");
}
```

**使用场景**：

```java
// 场景：查询多个服务，5秒内必须返回
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Instant deadline = Instant.now().plusSeconds(5);
    scope.fork(() -> callServiceA());
    scope.fork(() -> callServiceB());
    scope.fork(() -> callServiceC());

    try {
        scope.joinUntil(deadline);  // 5秒内必须完成
    } catch (TimeoutException e) {
        logger.warn("服务调用超时，取消未完成的任务");
        scope.close();  // 关闭 scope，所有子任务被取消
        throw new ServiceException("超时");
    }

    scope.throwIfFailed();
}
```

> ⚠️ 注意：`joinUntil()` 超时后，`scope` 仍处于打开状态，需要手动 `close()` 取消子任务。



**关键设计**：`StructuredTaskScope` 实现了 `AutoCloseable`，必须配合 `try-with-resources` 使用，确保子任务不会泄漏。

### 与 CompletableFuture 的对比

| 维度 | StructuredTaskScope | CompletableFuture |
|------|-------------------|-------------------|
| **任务管理** | 结构化，父任务等待子任务 | 扁平化，手动管理依赖链 |
| **异常传播** | `throwIfFailed()` 直接抛出 | `exceptionally()` / `handle()` 链式处理 |
| **取消机制** | 策略化自动取消 | 需手动 `cancel()` |
| **线程转储** | 显示任务父子关系 | 无法追踪任务来源 |
| **编程模型** | 同步代码风格（阻塞等待） | 链式异步回调 |
| **适用场景** | 多个并发子任务有聚合关系 | 异步流水线、事件驱动 |

```java
// CompletableFuture 方式（对比）
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> fetchUser(userId));
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> fetchOrder(orderId));
// 需要手动处理异常、手动取消
CompletableFuture.allOf(userFuture, orderFuture).join(); // 异常信息不清晰

// StructuredTaskScope 方式 — 更直观、更安全
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask = scope.fork(() -> fetchUser(userId));
    var orderTask = scope.fork(() -> fetchOrder(orderId));
    scope.join();
    scope.throwIfFailed(); // 异常传播清晰
}
```

### 与虚拟线程的配合

`StructuredTaskScope` 默认为每个 `fork` 的任务创建**虚拟线程**，天然适合大规模并发：

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // 每个 fork 自动使用虚拟线程，无需手动指定
    scope.fork(() -> callRemoteServiceA());  // 虚拟线程
    scope.fork(() -> callRemoteServiceB());  // 虚拟线程
    scope.fork(() -> callRemoteServiceC());  // 虚拟线程
    scope.join();
    scope.throwIfFailed();
}
```

**最佳实践**：`StructuredTaskScope` + 虚拟线程 = 用同步代码风格写出高性能、可管理的并发程序。

## 易错点/踩坑

- ❌ **忘记 try-with-resources**：`StructuredTaskScope` 必须在 `try` 块中使用，否则子任务可能泄漏
- ❌ **在 scope 外调用 Subtask.get()**：必须在 `scope.join()` 之后、`try` 块之内调用
- ❌ **JDK 版本混淆**：JDK 21 是预览版需 `--enable-preview`，JDK 23 才是正式版可直接使用
- ❌ **嵌套使用**：不要在一个 scope 内创建另一个 scope 的子任务，保持任务树的扁平结构

## 关联知识点

