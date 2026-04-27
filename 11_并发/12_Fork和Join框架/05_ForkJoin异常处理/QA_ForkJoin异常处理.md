
# ForkJoin 异常处理

## Q1：ForkJoinTask 的 join() 抛出什么异常？为什么不用 ExecutionException？

**A**：join() 将子任务的原始异常包装为 **RuntimeException** 重新抛出，而不是 ExecutionException。

原因：
- **简化递归异常处理**：递归调用栈深，如果每个子任务都用 ExecutionException，需要层层 throws 或 try-catch。用 RuntimeException 可以一路抛出到顶层统一处理
- **符合 ForkJoin 设计哲学**：分治任务中异常通常意味着整体失败，RuntimeException 减少异常处理的样板代码

代价：需要调用 `getException()` 才能获取原始异常，catch 块中拿到的不是原始异常类型。

```java
// 正确获取原始异常
try {
    task.join();
} catch (RuntimeException e) {
    Throwable cause = task.getException(); // 原始异常在这里
    if (cause instanceof IOException) {
        // 处理原始异常
    }
}
```

---

## Q2：get() 和 join() 在异常处理上有什么区别？

**A**：

| 特性 | join() | get() |
|------|--------|-------|
| 异常包装 | RuntimeException | ExecutionException |
| 中断支持 | ❌ 不支持 | ✅ 抛出 InterruptedException |
| 超时支持 | ❌ 不支持 | ✅ 抛出 TimeoutException |
| 适用位置 | ForkJoin 内部子任务 | 外部调用者 |
| 性能 | 轻量（无 try-catch 包装） | 稍重 |

```java
// get() 支持超时和安全中断
try {
    result = task.get(5, TimeUnit.SECONDS);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    // 安全中断处理
} catch (TimeoutException e) {
    // 超时处理
    task.cancel(true);
} catch (ExecutionException e) {
    // 获取原始异常
    Throwable cause = e.getCause();
}
```

---

## Q3：invokeAll 中一个子任务抛异常，其他子任务会怎样？

**A**：invokeAll 中某个子任务抛异常后，**其他子任务会被取消（cancel）**，这是与手动 fork 最大的区别。

```java
// invokeAll 内部行为
invokeAll(t1, t2);
// 等价于：
// t1.fork(); t2.fork();
// t1.join(); t2.join();
// 如果 t1 抛异常，t2 会被 cancel
```

**原理**：invokeAll 内部调用了 `ForkJoinTask.cancel()`，会向未完成的其他子任务发送取消信号。

**手动 fork 对比**：

```java
// 手动 fork：异常不会自动取消其他子任务
left.fork();
try {
    rightResult = right.compute();  // 如果这里抛异常，left 仍在运行
    leftResult = left.join();
} catch (Exception e) {
    // left 任务不会被自动取消，需手动处理
}
```

**最佳实践**：如果需要某个子任务失败不影响其他子任务，用手动 fork 模式，并在 catch 中主动处理。

---

## Q4：如何在递归任务中安全地处理异常？

**A**：推荐模式：**手动 fork + 统一异常处理 + 异常记录**：

```java
@Override
protected Long compute() {
    if (end - start <= THRESHOLD) {
        return sequentialSum();
    }

    int mid = start + (end - start) / 2;
    RecursiveTask<Long> left = new SubTask(start, mid);
    RecursiveTask<Long> right = new SubTask(mid, end);

    // 手动 fork，不自动传播异常
    left.fork();

    try {
        Long rightResult = right.compute();   // 先执行右侧
        Long leftResult = left.join();        // 再等待左侧
        return leftResult + rightResult;
    } catch (RuntimeException e) {
        // 记录异常，尝试返回默认值或重新抛出
        logger.warn("子任务异常，降级处理: {}", e.getMessage());
        return 0L;  // 降级策略
    }
}
```

关键点：
1. 先 fork 左侧，再 compute 右侧（保证右侧优先完成）
2. 用 try-catch 包裹 join()，不是包裹 compute()
3. 异常时选择降级策略（返回默认值）或重新抛出

---

## Q5：ForkJoinTask 如何检测任务是否异常完成？

**A**：通过 `isCompletedAbnormally()` 和 `getException()` 两个方法：

```java
ForkJoinTask<?> task = someTask;

// 方式1：检测是否异常完成
if (task.isCompletedAbnormally()) {
    Throwable ex = task.getException();
    // ex 就是原始抛出的异常（不是包装后的）
}

// 方式2：在 join 之前先检查
if (task.isDone() && !task.isCompletedNormally()) {
    // 任务已完成但异常
    throw task.getException();
}
```

`isCompletedAbnormally()` 返回 true 当且仅当 status == CANCELLED 或 EXCEPTIONAL。

---

```java
// 完整示例：异常安全的并行求和
ForkJoinPool pool = new ForkJoinPool();
SumTask task = new SumTask(array, 0, array.length);
Future<Long> future = pool.submit(task);

try {
    Long result = future.get(); // 外部用 get，内部子任务用 join
    System.out.println("Sum: " + result);
} catch (ExecutionException e) {
    // get() 抛出 ExecutionException
    Throwable cause = e.getCause();
    System.err.println("计算异常: " + cause.getMessage());
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    System.err.println("任务被中断");
}
```
