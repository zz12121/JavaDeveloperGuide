---
title: 线程方法
tags:
  - Java/并发
  - 问答
  - 对比型
module: 01_线程基础
created: 2026-04-18
---

# start vs run

## Q1：start() 和 run() 的核心区别？

**A**：

| 维度 | start() | run() |
|------|---------|-------|
| 作用 | 启动新线程 | 执行线程任务 |
| 线程数 | 创建新线程 | 不创建新线程 |
| 执行方式 | 异步执行 | 同步执行 |
| 调用次数 | 只能调用 1 次 | 可以多次调用 |
| 状态变化 | NEW → RUNNABLE | 无变化 |

---

## Q2：start() 的执行流程是什么？

**A**：
1. 检查线程状态是否为 NEW，否则抛出 `IllegalThreadStateException`
2. 将线程加入线程组
3. 调用 native 方法 `start0()` 向操作系统申请创建线程
4. 新线程启动后，JVM 自动调用 `run()` 方法
5. `start()` 方法立即返回，不等待 `run()` 执行完毕

---

## Q3：为什么不能直接调用 run()？

**A**：直接调用 `run()` 只是普通的方法调用，不会创建新线程：代码在当前线程中同步执行，不会进行线程上下文切换，没有多线程并发的效果。

---

## Q4：为什么不能多次调用 start()？

**A**：线程状态只能转换一次：NEW → RUNNABLE → TERMINATED。一旦线程启动或结束，就不能再次启动。多次调用 `start()` 会抛出 `IllegalThreadStateException`。

---

## Q5：调用 start() 后线程会立即执行吗？

**A**：不会立即执行。调用 `start()` 后线程进入 RUNNABLE 状态，等待操作系统调度分配 CPU 时间片。具体何时执行取决于线程调度器和系统负载。

---

## Q6：一个线程执行完后还能再次 start() 吗？

**A**：不能。线程执行完毕后进入 TERMINATED 状态，不能重新启动。如果需要再次执行任务，必须创建新的 Thread 对象。

---

## Q7：如果 run() 方法抛出异常会怎样？

**A**：如果 `start()` 启动的线程中 `run()` 抛出未捕获异常，线程会终止，异常可以通过 `Thread.setUncaughtExceptionHandler()` 捕获处理。如果是直接调用 `run()`，异常会在当前线程抛出，可以被 try-catch 捕获。

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


# sleep vs wait

## Q1：sleep() 和 wait() 的核心区别？

**A**：

|维度|sleep()|wait()|
|---|---|---|
|所属类|Thread（静态方法）|Object（实例方法）|
|锁释放|**不释放锁**|**释放锁**|
|调用条件|任意位置|必须在 synchronized 块内|
|唤醒方式|时间到自动唤醒|需 notify/notifyAll 或超时|
|使用场景|暂停执行|线程间协作|
|异常处理|抛出 InterruptedException|抛出 InterruptedException|

---

## Q2：sleep() 有什么特点？

**A**：
1. **不释放锁**：sleep 期间继续持有 synchronized 锁和其他锁
2. **自动唤醒**：指定时间到达后自动恢复 RUNNABLE 状态
3. **可中断**：可被 `interrupt()` 提前唤醒，抛出 `InterruptedException`
4. **静态方法**：作用于当前线程，与调用对象无关

---

## Q3：wait() 有什么特点？

**A**：
1. **必须配合 synchronized**：在 synchronized 块内调用，否则会抛异常
2. **释放锁**：调用 `wait()` 会释放对象的 monitor 锁
3. **进入等待队列**：线程进入对象的等待队列，状态变为 WAITING/TIMED_WAITING
4. **需要唤醒**：必须由其他线程调用 `notify()`/`notifyAll()` 唤醒，或超时自动唤醒
5. **唤醒后竞争锁**：被唤醒后需要重新竞争获取锁才能继续执行

---

## Q4：为什么 wait() 必须放在 while 循环中？

**A**：
```java
// 错误写法
if (!condition) {
    obj.wait();  // 被唤醒后可能条件仍不满足
}

// 正确写法
while (!condition) {
    obj.wait();  // 被唤醒后再次检查条件
}
```
原因：
1. **虚假唤醒**（Spurious Wakeup）：JVM 允许在没有 notify 的情况下唤醒线程
2. **多线程竞争**：`notifyAll()` 唤醒所有线程，只有一个能获取锁，其他需要继续等待

---

## Q5：wait() 和 sleep() 都会让出 CPU 吗？

**A**：都会让出 CPU。`sleep()` 让出 CPU 但持有锁；`wait()` 让出 CPU 且释放锁。两者都会进入阻塞状态，不再参与 CPU 调度，直到被唤醒或超时。

---

## Q6：notify() 和 notifyAll() 有什么区别？

**A**：`notify()` 随机唤醒一个等待该对象的线程；`notifyAll()` 唤醒所有等待该对象的线程。被唤醒的线程需要重新竞争锁，只有一个能获取锁继续执行，其他继续等待。

---

## Q7：sleep(0) 有什么作用？

**A**：`sleep(0)` 会让当前线程让出 CPU，进入就绪队列，让同优先级或更高优先级的线程有机会执行。但如果没有其他可运行线程，当前线程会立即再次获得 CPU。

---

## Q8：LockSupport.park() 和 wait() 有什么区别？

**A**：`park()` 不需要在 synchronized 块内调用，不会释放锁；`wait()` 必须在 synchronized 内，会释放锁。`park()` 使用许可（permit）机制，`unpark()` 可以先于 `park()` 调用；`wait()`/`notify()` 必须遵循先 wait 后 notify 的顺序。


# yield vs join

## Q1：yield() 和 join() 的核心区别？

**A**：

|特性|yield()|join()|
|---|---|---|
|方法类型|静态方法|实例方法|
|是否阻塞|非阻塞（只是提示）|阻塞（强制等待）|
|是否释放锁|不释放|释放（内部调用 wait）|
|实际效果|取决于调度器|等待线程终止|

---

## Q2：yield() 的具体行为？

**A**：`yield()` 是 `Thread` 类的静态方法，调用后只是向线程调度器提出建议，让出当前占用的 CPU 时间片给其他就绪状态的线程。**但这只是提示，调度器可能完全忽略它**。
关键点：
- 不释放任何锁
- 不保证线程切换
- 无法控制让给哪个线程
- 主要用于测试或调试场景

---

## Q3：join() 底层是怎么实现的？

**A**：`join()` 内部使用 `synchronized` 同步，调用 `wait()` 方法让当前线程等待。当目标线程执行完毕（`isAlive()` 返回 false）时，会自动调用 `notifyAll()` 唤醒等待线程。

---

## Q4：sleep() 和 yield() 有什么区别？

**A**：`sleep()` 会让线程进入**阻塞状态**，在指定时间内不会被调度；`yield()` 只是让出 CPU 提示，线程仍处于**就绪状态**，随时可能被再次调度。`sleep()` 更"可靠"，`yield()` 几乎没有实际保证。

---

## Q5：调用 interrupt() 会中断 join() 吗？

**A**：会。调用 `interrupt()` 会立即抛出 `InterruptedException`，停止阻塞状态。
```java
// yield()：让出 CPU 时间片，回到 RUNNABLE 重新竞争
Thread t = new Thread(() -> {
    for (int i = 0; i < 5; i++) {
        Thread.yield();  // 每次循环让出 CPU，但不保证其他线程一定执行
        System.out.println("Thread: " + i);
    }
});

// join()：等待目标线程执行完毕
Thread worker = new Thread(() -> {
    System.out.println("工作线程开始");
    try { Thread.sleep(2000); } catch (InterruptedException e) {}
    System.out.println("工作线程结束");
});
worker.start();
worker.join();  // 主线程阻塞，等待 worker 完成
// worker.join(3000);  // 最多等 3 秒
System.out.println("主线程继续");
```

---

## Q6：`join()` 有超时参数，超时后发生什么？会不会抛异常？

**A**：`join(long millis)` 超时后**不会抛出异常**，调用线程直接返回继续执行，目标线程继续在后台运行（不会被终止）。

```java
Thread worker = new Thread(() -> {
    try {
        Thread.sleep(5000);  // 模拟耗时 5 秒
        System.out.println("工作线程完成");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});
worker.start();

// 最多等 2 秒
worker.join(2000);

// 2 秒后不管 worker 是否完成，主线程都会继续执行
// 此时 worker 仍在后台运行
System.out.println("主线程超时返回，worker 状态: " + worker.getState()); // TIMED_WAITING

// 判断 worker 是否在 join 超时后完成（可用于超时检测）
if (worker.isAlive()) {
    System.out.println("worker 仍在运行，可选择中断或等待");
    worker.interrupt(); // 可选：主动中断
}
```

> **注意**：如果等待期间被 `interrupt()`，`join()` 会抛出 `InterruptedException`。但单纯超时**不抛异常**，这与 `wait(timeout)` 行为一致。

## 关联知识点