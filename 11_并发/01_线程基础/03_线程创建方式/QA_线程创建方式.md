---
title: 线程创建方式
tags:
  - Java/并发
  - 问答
  - 场景型
module: 01_线程基础
created: 2026-04-18
---

# 线程创建方式（Thread / Runnable / Callable + Future / 线程池）

## Q1：Java 有哪几种创建线程的方式？

**A**：四种方式：

1. **继承 Thread 类**：重写 `run()` 方法，直接调用 `start()`
2. **实现 Runnable 接口**：实现 `run()` 方法，解耦线程定义与启动
3. **实现 Callable 接口 + FutureTask**：有返回值，可抛异常
4. **线程池**：`Executors` 创建，复用线程，控制并发数

---

## Q2：Callable 和 Runnable 有什么区别？

**A**：

| 对比项 | Runnable | Callable |
|--------|----------|----------|
| 返回值 | 无 | 有（通过 `Future` 获取） |
| 异常 | 不能抛受检异常 | 可以抛受检异常 |
| 方法签名 | `void run()` | `V call() throws Exception` |
| 使用方式 | `Thread` 或线程池 | 配合 `FutureTask` 或线程池 `submit()` |

---

## Q3：为什么不推荐直接 new Thread？

**A**：每次创建线程都需要分配栈空间、创建系统线程，开销大；大量创建会导致 OOM。线程池可复用线程，减少开销，并提供任务管理（拒绝策略、超时等）。

---

## Q4：线程池的核心线程会被回收吗？

**A**：默认情况下，核心线程不会被回收。设置 `allowCoreThreadTimeOut(true)` 后，核心线程也会在 `keepAliveTime` 时间后被回收。

---

## Q5：submit() 和 execute() 的区别？

**A**：`submit()` 可提交 `Callable`/`Runnable`，返回 `Future`；`execute()` 只能提交 `Runnable`，无返回值。
```java
// 方式1：继承 Thread
new Thread() {
    @Override
    public void run() { System.out.println("Thread"); }
}.start();

// 方式2：实现 Runnable
new Thread(() -> System.out.println("Runnable")).start();

// 方式3：Callable + FutureTask
FutureTask<String> task = new FutureTask<>(() -> "result");
new Thread(task).start();
String result = task.get();  // "result"

// 方式4：线程池（推荐）
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> future = pool.submit(() -> "pool result");
```


## 关联知识点
