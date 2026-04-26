
# newSingleThreadExecutor

## Q1：newSingleThreadExecutor 和 newFixedThreadPool(1) 有什么区别？

**A**：

参数完全相同（core=1, max=1, 无界队列），区别在返回类型：

- **newSingleThreadExecutor**：返回 `FinalizableDelegatedExecutorService`，无法强转为 ThreadPoolExecutor，**不能运行时修改参数**
- **newFixedThreadPool(1)**：返回 ThreadPoolExecutor，可以强转后 `setCorePoolSize(5)` 修改线程数

包装目的是**防止误操作**，确保始终是单线程。

---

## Q2：单线程池中一个任务抛异常会影响后续任务吗？

**A**：**不会**。ThreadPoolExecutor 在 runWorker 中有 try-catch：

```java
try {
    task.run();
} catch (Throwable ex) {
    afterExecute(task, ex); // 记录异常
} finally {
    task = null; // 继续从队列取下一个任务
}
```

一个任务异常不会导致线程退出，后续任务正常执行。但异常会被"吞掉"（execute 模式），建议用 submit + future.get() 捕获。

