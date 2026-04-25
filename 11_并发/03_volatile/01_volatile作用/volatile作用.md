---
title: volatile作用
tags:
  - Java/并发
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile作用（可见性 + 有序性，不保证原子性）

## 先说结论

volatile 是 Java 提供的轻量级同步机制，能保证被修饰变量的**可见性**和**有序性（禁止指令重排序）**，但**不保证原子性**。它是并发编程中理解 JMM（Java 内存模型）的基石关键字。

## 深度解析

### 可见性

在 JMM 中，每个线程有自己的工作内存（CPU 缓存），线程对普通变量的修改不会立即同步到主内存，其他线程读到的可能是过期值。volatile 修饰的变量，**写操作会强制刷新到主内存，读操作会从主内存重新加载**，从而保证所有线程看到的是最新值。

### 有序性

编译器和 CPU 为了优化性能，会对指令进行重排序。volatile 通过插入**内存屏障（Memory Barrier）** 来禁止特定类型的重排序，保证 volatile 变量的读写操作不会被重排到屏障的另一侧。

### 为什么不保证原子性

volatile 仅仅保证单次读/写操作的可见性，但对于复合操作（如 `i++`、`if-then-act`）无法保证原子性。`i++` 实际是"读-改-写"三步操作，volatile 不能将这三步锁定为一个不可分割的整体。

### 与 synchronized 的对比

| 特性     | volatile             | synchronized       |
| -------- | -------------------- | ------------------ |
| 可见性   | 保证                 | 保证               |
| 有序性   | 保证（禁止重排序）   | 保证（进入/退出屏障） |
| 原子性   | 不保证               | 保证               |
| 阻塞     | 不阻塞               | 会阻塞             |
| 适用场景 | 单一变量的读写       | 临界区的保护       |

## 易错点/踩坑

- ❌ 认为 volatile 可以替代 synchronized，在所有并发场景下使用
- ❌ 用 volatile 修饰 `i++` 操作，以为能解决并发计数问题
- ✅ volatile 适用于"一个线程写、多个线程读"的状态标志场景
- ✅ 需要原子性时应使用 `AtomicInteger`、`synchronized` 或 `Lock`

## 代码示例

```java
// volatile 修改变量，保证可见性但不保证原子性
public class VolatileVisibilityDemo {
    private volatile boolean running = true;

    public void stopTask() {
        running = false;  // 写入主内存，其他线程立即可见
    }

    public void doWork() {
        while (running) {  // 每次从主内存重新读取
            // 执行业务逻辑
        }
    }
}
```

```java
// volatile 不保证原子性的经典反例
public class VolatileAtomicDemo {
    private volatile int count = 0;

    public void increment() {
        count++;  // 非原子操作：读 -> 改 -> 写
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileAtomicDemo demo = new VolatileAtomicDemo();
        Runnable task = () -> {
            for (int i = 0; i < 10000; i++) demo.increment();
        };
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join(); t2.join();
        // 期望 20000，实际结果通常小于 20000
        System.out.println("最终 count = " + demo.count);
    }
}
```

## 图解/流程

```
普通变量：
  线程A (CPU缓存)          主内存           线程B (CPU缓存)
  ┌──────────┐          ┌──────────┐      ┌──────────┐
  │ count=5  │ ──修改──>│ count=5  │      │ count=5  │
  │ count=6  │          │ (未刷新) │      │ (读旧值) │
  └──────────┘          └──────────┘      └──────────┘

volatile变量：
  线程A (CPU缓存)          主内存           线程B (CPU缓存)
  ┌──────────┐          ┌──────────┐      ┌──────────┐
  │ count=6  │ ══强制══>│ count=6  │══强制═>│ count=6  │
  │          │   刷新    │          │  重新  │          │
  └──────────┘          └──────────┘   加载  └──────────┘
```

## 关联知识点
