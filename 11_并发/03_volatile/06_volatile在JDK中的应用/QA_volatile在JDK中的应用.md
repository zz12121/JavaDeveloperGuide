---
title: volatile在JDK中的应用
tags:
  - Java/并发
  - 问答
  - 常考
module: 03_volatile
created: 2026-04-18
---

# volatile在JDK中的应用（DCL单例/ConcurrentHashMap/ReentrantLock）

## Q1：ConcurrentHashMap 中哪些字段用了 volatile？为什么？

**A**：ConcurrentHashMap 的 Node 类中 `val` 和 `next` 两个字段用了 volatile：

```java
static class Node<K,V> {
    volatile V val;
    volatile Node<K,V> next;
}
```

**原因**：ConcurrentHashMap 的 `get()` 操作是无锁的（不像 put 需要加锁），为了让 get 方法能读到最新写入的值，val 和 next 必须用 volatile 保证可见性。而 put 操作通过 synchronized + CAS 来保证原子性和可见性。

---

## Q2：AQS 中的 volatile state 是如何工作的？

**A**：`AbstractQueuedSynchronizer` 中的 `state` 是 volatile 修饰的 int：

- **写操作**：通过 `Unsafe.compareAndSwapInt()` 原子修改，CAS 本身具有原子性
- **读操作**：通过 volatile 的可见性保证，所有线程都能读到最新 state
- **语义**：`volatile` 保证可见性，`CAS` 保证原子性，两者配合实现了无锁的同步状态管理

ReentrantLock 的 `lock()` 通过 CAS 将 state 从 0 改为 1 表示获取锁，`unlock()` 将 state 改为 0 表示释放锁。

---

## Q3：ThreadPoolExecutor 的 ctl 字段为什么用 volatile？

**A**：ctl 是一个 `volatile int`，高 3 位表示线程池状态（RUNNING/SHUTDOWN/STOP/TIDYING/TERMINATED），低 29 位表示工作线程数量。volatile 保证：
1. `shutdown()` 修改状态后，工作线程通过 `ctl` 能立即感知
2. `execute()` 检查线程数时能读到最新值
3. 多个线程并发提交任务时，ctl 的读取始终是最新的

## 关联知识点
