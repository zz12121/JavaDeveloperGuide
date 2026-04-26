
# 线程池异常处理

## Q1：execute 和 submit 的异常处理有什么区别？

**A**：

- **execute**：异常被 runWorker 的 try-catch 捕获，不会影响后续任务，但异常默认被"吞掉"（可通过 afterExecute 钩子处理）
- **submit**：异常封装在 Future 中，调用 `future.get()` 时抛 `ExecutionException`

submit 不调用 get() 时异常也会被吞掉。

---

## Q2：为什么 UncaughtExceptionHandler 对 execute 无效？

**A**：ThreadPoolExecutor.runWorker() 中有 try-catch 包裹 `task.run()`：

```java
try {
    task.run();
} catch (Throwable ex) {
    afterExecute(task, ex); // 异常被这里捕获
}
```

异常不会传播到 Thread 的 UncaughtExceptionHandler，而是被 afterExecute 接收。

---

## Q3：线程池异常处理的最佳实践？

**A**：

1. **优先用 submit**，配合 future.get() 捕获异常
2. **自定义 afterExecute**，全局记录异常日志
3. **周期任务必须 try-catch**，否则后续调度被取消
4. **CompletableFuture.exceptionally()**，链式处理异常

