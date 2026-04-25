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

# start vs run

## Q1：start() 和 run() 的核心区别？

**A**：

|维度|start()|run()|
|---|---|---|
|作用|启动新线程|执行线程任务|
|线程数|创建新线程|不创建新线程|
|执行方式|异步执行|同步执行|
|调用次数|只能调用 1 次|可以多次调用|
|状态变化|NEW → RUNNABLE|无变化|

---

## Q2：start() 的执行流程是什么？

**A**：

1. 检查线程状态是否为 NEW，否则抛出 `IllegalThreadStateException`
2. 将线程加入线程组
3. 调用 native 方法 `start0()` 向操作系统申请创建线程
4. 新线程启动后，JVM 自动调用 `run()` 方法
5. `start()` 方法立即返回，不等待 `run()` 执行完毕

---

## Q3：为什么不能直接调用 run()？

**A**：直接调用 `run()` 只是普通的方法调用，不会创建新线程：代码在当前线程中同步执行，不会进行线程上下文切换，没有多线程并发的效果。

---

## Q4：为什么不能多次调用 start()？

**A**：线程状态只能转换一次：NEW → RUNNABLE → TERMINATED。一旦线程启动或结束，就不能再次启动。多次调用 `start()` 会抛出 `IllegalThreadStateException`。

---

## Q5：调用 start() 后线程会立即执行吗？

**A**：不会立即执行。调用 `start()` 后线程进入 RUNNABLE 状态，等待操作系统调度分配 CPU 时间片。具体何时执行取决于线程调度器和系统负载。

---

---

## Q6：线程池的四种拒绝策略分别是什么？

**A**：当线程池**队列已满且线程数达到最大值**时触发拒绝策略：

| 策略 | 行为 | 特点 |
|------|------|------|
| `AbortPolicy`（默认） | 抛出 `RejectedExecutionException` | 任务提交方必须处理异常 |
| `CallerRunsPolicy` | 调用者线程直接执行任务 | 相当于降速，不丢任务 |
| `DiscardPolicy` | 静默丢弃新任务 | 适合允许丢失的场景 |
| `DiscardOldestPolicy` | 丢弃队列头部最老的任务，再重试提交 | 优先保留新任务 |

> **生产环境建议**：使用 `CallerRunsPolicy` 或自定义策略（记录日志、存入数据库等），避免无声丢失任务。

---

## Q7：`shutdown()` 和 `shutdownNow()` 有什么区别？

**A**：

| 方法 | 行为 | 已提交任务 | 正在执行任务 |
|------|------|-----------|------------|
| `shutdown()` | 温和关闭 | 继续执行完 | 不中断 |
| `shutdownNow()` | 立即关闭 | 返回未执行列表 | 尝试中断（interrupt）|

```java
// 推荐的优雅关闭写法
pool.shutdown();
try {
    if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
        pool.shutdownNow();  // 超时再强制关闭
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}
```

> **注意**：`shutdownNow()` 只是向正在执行的任务**发送中断信号**，任务是否响应取决于任务内部是否正确处理中断。


**A**：不能。线程执行完毕后进入 TERMINATED 状态，不能重新启动。如果需要再次执行任务，必须创建新的 Thread 对象。

---

## Q7：如果 run() 方法抛出异常会怎样？


 **A**：如果 `start()` 启动的线程中 `run()` 抛出未捕获异常，线程会终止，异常可以通过 `Thread.setUncaughtExceptionHandler()` 捕获处理。如果是直接调用 `run()`，异常会在当前线程抛出，可以被 try-catch 捕获。
```java
// start()：启动新线程，异步执行
Thread t = new Thread(() -> System.out.println("新线程: " + Thread.currentThread().getName()));
t.start();   // NEW → RUNNABLE，新线程执行 run()
System.out.println("主线程继续执行");

// run()：普通方法调用，在当前线程同步执行
Thread t2 = new Thread(() -> System.out.println("当前线程: " + Thread.currentThread().getName()));
t2.run();    // 没有创建新线程！在主线程中执行

// 多次调用 start()：IllegalThreadStateException
t.start();   // 抛异常，线程已启动
```

