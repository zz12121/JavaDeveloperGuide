---
title: 线程优先级
tags:
  - Java/并发
  - 原理型
  - 问答
module: 01_线程基础
created: 2026-04-18
---

# 线程优先级（1-10级，优先级高不代表一定先执行）

## Q1：Java 线程优先级的范围？

**A**：Java 线程优先级使用 1~10 的整数表示：
- `Thread.MIN_PRIORITY = 1`：最低优先级
- `Thread.NORM_PRIORITY = 5`：默认优先级
- `Thread.MAX_PRIORITY = 10`：最高优先级

主线程默认优先级为 5，创建的子线程会继承父线程的优先级。

---

## Q2：高优先级的线程一定先执行吗？

**A**：**不一定**。线程优先级只是给调度器的一个**提示**，表示"这个线程相对更重要"：
- 调度器可能完全忽略优先级设置
- 高优先级不代表一定先执行，只是**获取 CPU 时间片的概率更高**
- 在多核 CPU 上，优先级影响可能更不明显

---

## Q3：为什么说依赖优先级做业务排序是不可靠的？

**A**：Java 线程优先级依赖于底层操作系统线程调度实现。不同操作系统对优先级的映射和处理方式不同，可能被忽略或映射到不同范围。在 Linux 上，JVM 的 10 级优先级会被映射到内核的较小范围，导致区分度降低。

---

## Q4：子线程的优先级默认值是什么？

**A**：子线程会继承父线程的优先级。如果主线程是 `NORM_PRIORITY(5)`，则子线程默认优先级也是 5。可以在 `start()` 之前通过 `setPriority()` 修改。

---

## Q5：如何让低优先级线程也能公平执行？

**A**：
1. 使用 `Thread.yield()` 让出 CPU
2. 使用信号量/计数器协调执行顺序
3. 使用 `wait()/notify()` 机制
4. 使用阻塞队列等并发工具协调线程
---
```java
Thread high = new Thread(() -> { for (int i = 0; i < 5; i++) System.out.println("高: " + i); });
Thread low = new Thread(() -> { for (int i = 0; i < 5; i++) System.out.println("低: " + i); });

high.setPriority(Thread.MAX_PRIORITY);  // 10
low.setPriority(Thread.MIN_PRIORITY);    // 1

high.start();
low.start();
// 结果不保证高优先级一定先执行完！
```


## 关联知识点
