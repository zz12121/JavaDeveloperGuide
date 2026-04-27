---
id: qa_208
title: 虚拟线程使用
tags:
  - Java/并发
  - 问答
  - 加分
  - 场景型
module: "19_虚拟线程"
created: 2026-04-18
---

# 虚拟线程使用

## Q1：Java 21 中有哪些创建虚拟线程的方式？

**A**：

Java 21 提供了四种创建虚拟线程的方式：

1. **`Thread.startVirtualThread(Runnable)`**：最简单，一行代码创建并启动
2. **`Thread.ofVirtual().start(Runnable)`**：Builder 模式，可配置线程名和异常处理器
3. **`Executors.newVirtualThreadPerTaskExecutor()`**：返回 ExecutorService，**推荐方式**，与现有 API 兼容
4. **自定义 ThreadFactory**：`Thread.ofVirtual().factory()` 创建工厂，配合 ThreadPoolExecutor 使用

日常开发推荐方式三，因为返回的是标准的 `ExecutorService`，可以直接替换现有的线程池。

---

## Q2：为什么推荐使用 Executors.newVirtualThreadPerTaskExecutor()？

**A**：

三个原因：

1. **API 兼容性**：返回 `ExecutorService`，可以直接替换项目中的线程池，改造成本极低
2. **生命周期管理**：支持 `try-with-resources`，确保所有任务完成后自动关闭
3. **语义正确**：虚拟线程本身非常轻量，不需要像平台线程那样复用，每任务一线程是正确做法

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i -> {
        executor.submit(() -> handleRequest(i));
    });
}
```

---

## Q3：如何判断当前代码运行在虚拟线程还是平台线程中？

**A**：

```java
Thread.currentThread().isVirtual();  // true = 虚拟线程，false = 平台线程
```

这在混合使用虚拟线程和平台线程的场景下非常有用，例如调试和日志记录。

---

## Q4：虚拟线程能和 CompletableFuture 配合使用吗？

**A**：

完全可以。`CompletableFuture.supplyAsync()` 接收 `Executor` 参数，直接传入虚拟线程执行器即可：

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> future = CompletableFuture
        .supplyAsync(() -> fetchData(), executor)
        .thenApply(data -> process(data))
        .thenAccept(result -> System.out.println(result));
    future.join();
}
```

这比使用 `ForkJoinPool.commonPool()` 更高效，因为虚拟线程在遇到阻塞时不会浪费载体线程。


