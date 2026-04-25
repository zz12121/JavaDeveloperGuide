---
title: synchronized特性
tags:
  - Java/并发
  - 原理型
module: 04_synchronized
created: 2026-04-18
difficulty: 中
---

# synchronized特性（可重入/不可中断/阻塞式/独享/互斥）

## 先说结论

synchronized 是 JVM 内置的互斥同步机制，具备**可重入、不可中断、阻塞式、独享、互斥**五大特性。它同时保证原子性、可见性和有序性，是 Java 并发编程中最基础的锁实现。

## 深度解析

### 五大特性

1. **可重入**：同一线程可多次获取同一把锁，JVM 通过记录持有线程 ID 和重入次数实现
2. **不可中断**：线程等待 synchronized 锁时无法响应 `Thread.interrupt()`，只能等待获取锁或抛异常
3. **阻塞式**：竞争不到锁的线程进入 BLOCKED 状态，涉及用户态→内核态切换
4. **独享**：同一时刻只有一个线程能持有该锁
5. **互斥**：其他线程必须等待当前线程释放锁后才能竞争

### JVM 保证的三大语义

- **原子性**：同步代码块同一时刻只允许一个线程执行
- **可见性**：unlock 前刷新共享变量到主内存，lock 时从主内存重新读取
- **有序性**：同步代码块内的指令不会被重排到块外

### vs ReentrantLock

| 特性     | synchronized | ReentrantLock |
| -------- | ------------ | ------------- |
| 可中断   | ❌            | ✅ lockInterruptibly |
| 公平锁   | ❌ 非公平     | ✅ 可选公平     |
| 条件变量 | 单一 wait/notify | 多个 Condition |
| 释放锁   | 自动（作用域结束） | 手动 unlock    |

## 易错点/踩坑

- ❌ 认为可重入意味着不同线程可以重入
- ✅ 可重入仅限**同一线程**
- ❌ 以为 synchronized 等待状态是 WAITING
- ✅ 竞争锁是 BLOCKED，`Object.wait()` 才是 WAITING
- ❌ 以为 synchronized 不可中断就永远无法停止
- ✅ JVM 可以通过 safepoint 机制中止等待

## 代码示例

```java
public class ReentrantDemo {
    private int count = 0;

    public synchronized void methodA() {
        count++;
        methodB(); // 同一线程重入，不会死锁
    }

    public synchronized void methodB() {
        count++;
    }

    public static void main(String[] args) throws InterruptedException {
        ReentrantDemo demo = new ReentrantDemo();
        Thread t = new Thread(() -> {
            demo.methodA();
            System.out.println("count = " + demo.count); // count = 2
        });
        t.start();
        t.join();
    }
}
```

## 图解/流程

```
synchronized 锁获取流程：
        │
        ├── 锁空闲？ ──Yes──▶ 获取锁，执行同步代码
        │
        └── 锁已被占用？
                │
                ├── 占用者是本线程？ ──Yes──▶ 重入次数+1，继续执行
                │
                └── 占用者是其他线程？ ──▶ 进入 BLOCKED，等待
                                                    │
                                              锁释放后被唤醒
                                                    │
                                              重新竞争锁
```

## 关联知识点
