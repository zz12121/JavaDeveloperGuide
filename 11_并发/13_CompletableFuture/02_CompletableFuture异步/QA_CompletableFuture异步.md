---
id: qa_142
title: CompletableFuture异步
tags:
  - Java/并发
  - 问答
  - 常考
  - 原理型
module: "13_CompletableFuture"
created: 2026-04-18
---

# CompletableFuture异步

## Q1：thenApply 和 thenApplyAsync 有什么区别？

**A**：
- **thenApply**：在上一个阶段**完成的同一线程**中执行，是同步调用
- **thenApplyAsync**：在 commonPool 或**指定 Executor 的新线程**中执行，是异步调用

区别本质：**回调在哪个线程执行**。thenApply 不切换线程（减少开销），thenApplyAsync 切换线程（解耦、避免阻塞）。

---

## Q2：什么时候用带 Async 后缀的方法？

**A**：
- **需要 Async**：回调中有 I/O 阻塞（数据库、HTTP）、耗时长（> 1ms）、需要独立线程避免阻塞前序阶段、需要在不同线程池执行
- **不需要 Async**：回调轻量（纯 CPU 计算 < 1ms）、需要保持线程上下文（如 ThreadLocal）、避免不必要的线程切换开销

实践原则：默认不带 Async，遇到阻塞或耗时操作时加 Async 并指定独立线程池。

---

## Q3：不带 Async 的回调会阻塞什么？

**A**：不带 Async 的回调在上一个阶段完成的线程中**同步执行**，意味着：
- 如果前序阶段用的是 commonPool 的工作线程，回调执行期间该线程被占用
- 如果回调本身是阻塞 I/O，commonPool 的工作线程被阻塞
- 后续所有依赖的回调都要等这个阻塞完成

所以**不要在不带 Async 的回调中做阻塞操作**。



> **代码示例：thenApply vs thenApplyAsync 的线程差异**

```java
ExecutorService bizPool = Executors.newFixedThreadPool(4);

CompletableFuture.supplyAsync(() -> {
    return "hello"; // 在 ForkJoinPool.commonPool 执行
}).thenApply(s -> {
    // ⚠️ 不带 Async：仍在 commonPool 线程中同步执行
    return s.toUpperCase(); // 如果这里阻塞，commonPool 线程被占用
}).thenApplyAsync(s -> {
    // ✅ 带 Async：切换到 bizPool 线程执行
    return s + " WORLD";
}, bizPool);

// 最佳实践：回调中有阻塞操作时，必须用 Async + 独立线程池
```

