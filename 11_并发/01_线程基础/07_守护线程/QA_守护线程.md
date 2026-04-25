---
title: 守护线程
tags:
  - Java/并发
  - 问答
  - 原理型
module: 01_线程基础
created: 2026-04-18
---

# 守护线程（JVM退出时不等待守护线程，常用于GC线程）

## Q1：什么是守护线程？

**A**：守护线程（Daemon Thread）是 JVM 的后台服务线程，核心特性是：**当所有用户线程（非守护线程）都结束时，JVM 会立即退出，不会等待守护线程执行完毕**。
- **用途**：提供后台服务，如垃圾回收、内存监控、连接池维护等
- **设置方式**：`thread.setDaemon(true)`，必须在 `start()` 之前调用

---

## Q2：守护线程和用户线程的区别？

**A**：

| 对比项 | 用户线程 | 守护线程 |
|--------|----------|----------|
| 生命周期 | JVM 会等待其执行完毕 | JVM 退出时不等待 |
| finally 块 | 正常执行 | **不保证执行** |
| 子线程继承 | 默认为用户线程 | 继承父线程的守护属性 |
| 典型应用 | 业务逻辑处理 | GC 线程、监控线程 |

---

## Q3：常见使用场景有哪些？

**A**：
1. **垃圾回收线程**：JVM 自带的 GC 线程是典型的守护线程
2. **日志写入线程**：后台异步写入日志
3. **监控线程**：监控系统状态、收集指标
4. **定时任务线程**：执行周期性的后台清理任务

---

## Q4：为什么 GC 线程要设计为守护线程？

**A**：GC 是后台服务，不应该影响用户程序的正常退出。如果 GC 是用户线程，当用户程序结束时，JVM 需要等待 GC 完成才能退出，这是不合理的。设计为守护线程后，JVM 可以随时退出。

---

## Q5：守护线程的 finally 块为什么不保证执行？

**A**：当最后一个用户线程结束时，JVM 会立即开始关闭过程，不会等待守护线程的 finally 块执行完成。所以不应该在守护线程中执行关键清理操作。

---

## Q6：子线程的守护属性如何确定？

**A**：子线程的守护属性默认与父线程相同。如果父线程是守护线程，则子线程也是守护线程；如果父线程是用户线程，则子线程也是用户线程。这个属性在 `Thread.init()` 方法中被设置。
```java
// 守护线程：用户线程全部结束后自动销毁
Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("守护线程运行中...");
        try { Thread.sleep(1000); } catch (InterruptedException e) { break; }
    }
});
daemon.setDaemon(true);  // 必须在 start() 之前设置
daemon.start();

Thread.sleep(3000);  // 主线程（用户线程）结束
// 守护线程自动销毁，不会继续运行
```


---

## Q7：如何在 JVM 退出时保证守护线程的资源被正确释放？

**A**：使用 `Runtime.getRuntime().addShutdownHook()` 注册 JVM 关闭钩子，在钩子线程中执行清理操作：

```java
// 注册关闭钩子
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // 关闭连接池、释放文件句柄、刷日志缓冲等
    System.out.println("执行资源清理...");
    connectionPool.shutdown();
    logBuffer.flush();
}, "cleanup-hook"));
```

**原理**：ShutdownHook 本质是注册了一个**用户线程**，JVM 在退出时会等待所有 ShutdownHook 执行完毕，因此可以保证清理操作一定执行（强制 kill -9 除外）。

**注意事项**：
- 守护线程中**不应依赖 finally 块**做关键清理，应使用 ShutdownHook 代替
- 可以注册多个 ShutdownHook，它们并发执行，顺序不保证
- ShutdownHook 中禁止调用 `System.exit()`

## 关联知识点
