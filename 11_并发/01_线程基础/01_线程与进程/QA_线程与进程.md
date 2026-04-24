---
title: 线程与进程
tags:
  - Java/并发
  - 原理型
  - 问答
module: 01_线程基础
created: 2026-04-18
---

# 线程与进程（进程是资源分配单位，线程是CPU调度单位，共享进程资源）

## Q1：进程和线程有什么区别？

**A**：

| 维度 | 进程 | 线程 |
|------|------|------|
| 定义 | 资源分配的基本单位 | CPU调度的基本单位（轻量级进程） |
| 资源 | 独立地址空间 | 共享进程资源 |
| 开销 | 创建/切换开销大 | 开销小 |
| 通信 | 需要 IPC 机制 | 共享内存直接通信 |

---

## Q2：JVM 中哪些内存区域是线程共享的，哪些是线程私有的？

**A**：
- **线程共享**：堆内存、方法区（元空间）
- **线程私有**：虚拟机栈、本地方法栈、程序计数器

> 面试一句话总结：线程共享的是"对象生存的地方"（堆+方法区），私有的是"执行需要的上下文"（栈+PC）。

---

## Q3：线程间如何通信？

**A**：同一进程的线程通过共享变量通信，但需注意线程安全问题（如使用 synchronized、Lock 等同步机制）。

---

## Q4：为什么多线程比多进程更轻量？

**A**：线程共享进程的堆和方法区，创建线程只需分配少量私有资源（栈、PC寄存器），无需分配独立地址空间。

> **代码示例：线程创建的两种方式**
```java
// 方式1：继承 Thread
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " 运行中");
    }
}
new MyThread().start();

// 方式2：实现 Runnable（推荐）
Thread t = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " 运行中");
}, "worker-1");
t.start();
```

## 关联知识点
