
# execute vs submit

## 核心结论

`execute` 执行 Runnable 无返回值，异常直接抛到调用方。`submit` 可以提交 Runnable/Callable，返回 Future，异常封装在 Future 中。

## 深度解析

### 对比

| 维度 | execute | submit |
|------|---------|--------|
| 参数 | Runnable | Runnable / Callable |
| 返回值 | void | Future / Future<V> |
| 异常处理 | 直接抛到调用方 | 封装在 Future 中，get() 时抛 |
| 方法来源 | Executor 接口 | ExecutorService 接口 |
| 位置 | ThreadPoolExecutor | AbstractExecutorService |

### 异常处理差异

```java
// execute：异常直接抛出
executor.execute(() -> {
    throw new RuntimeException("异常"); // 抛到调用线程的 UncaughtExceptionHandler
});

// submit：异常封装在 Future 中
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("异常"); // 不会直接抛出
});
future.get(); // 此时抛 ExecutionException
```

### submit 三种重载

```java
// 1. Runnable（无返回值）
Future<?> submit(Runnable task);

// 2. Runnable + 结果
<T> Future<T> submit(Runnable task, T result);
// future.get() 返回 result

// 3. Callable（有返回值）
<T> Future<T> submit(Callable<T> task);
// future.get() 返回 call() 的返回值
```

### 使用建议

```java
// 不需要返回值 → execute
executor.execute(() -> doSomething());

// 需要返回值 → submit
Future<Integer> future = executor.submit(() -> compute());
Integer result = future.get(3, TimeUnit.SECONDS);

// 需要异常处理 → submit + try-catch Future
Future<?> f = executor.submit(() -> riskyOperation());
try {
    f.get();
} catch (ExecutionException e) {
    log.error("任务执行失败", e.getCause());
}
```

## 易错点与踩坑

### 1. submit 异常被"吞掉"的经典问题
```java
// 最常见的坑：submit 后不调用 get()，异常完全丢失
executor.submit(() -> {
    throw new RuntimeException("异常"); // 被封装在 Future 中，无人知晓
});
// 如果不调用 future.get()，异常永远不会被抛出
// 日志中看不到任何错误，任务静默失败
```

### 2. submit(Runnable) 返回值的误解
```java
Future<?> f = executor.submit(() -> {
    System.out.println("running");
});
Object result = f.get(); // 返回 null，不是任务的任何返回值

// 如果需要返回值，必须用 Callable
Future<Integer> f2 = executor.submit(() -> 42);
Integer r = f2.get(); // 返回 42
```

### 3. execute 异常不是"完全不处理"
```java
// execute 中的异常会被 ThreadPoolExecutor.runWorker() 的 try-catch 捕获
// 捕获后会调用 afterExecute(task, throwable)
// 但如果 afterExecute 没有被覆写 → 异常真的被吞了
// 且异常不会导致线程池中的线程死亡，后续任务继续执行
```

### 4. Future.cancel(true) 不保证任务停止
```java
future.cancel(true); // 只是发送中断信号
// 如果任务中没有检查 Thread.interrupted()，cancel 无效
// 类似 shutdownNow 的中断机制
```
## 关联知识点
