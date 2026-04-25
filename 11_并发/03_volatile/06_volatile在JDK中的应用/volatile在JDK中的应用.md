---
title: volatile在JDK中的应用
tags:
  - Java/并发
  - 源码型
module: 03_volatile
created: 2026-04-18
---

# volatile在JDK中的应用（DCL单例/ConcurrentHashMap/ReentrantLock）

## 先说结论

volatile 在 JDK 源码中被广泛使用，经典案例包括：ConcurrentHashMap 的 Node.val/next 字段、ReentrantLock 的 AbstractQueuedSynchronizer.state 字段、DCL 单例的 instance 引用。这些场景的共同特点是：变量被多线程读写且需要立即可见，但不涉及需要互斥的复合操作。

## 深度解析

### ConcurrentHashMap

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;       // 保证读线程能看到最新值
    volatile Node<K,V> next; // 保证链表结构的可见性
}
```

Node 的 val 和 next 用 volatile 修饰，因为 ConcurrentHashMap 的读操作（get）是无锁的，需要通过 volatile 保证读到最新值。

### ReentrantLock（AQS）

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;  // 同步状态，CAS + volatile
}
```

state 是 AQS 的核心字段，通过 CAS + volatile 实现：修改用 CAS 保证原子性，读取通过 volatile 保证可见性。

### Thread

```java
public class Thread implements Runnable {
    volatile ThreadLocal.ThreadLocalMap threadLocals;
    Object threadLocals = null;  // 不需要 volatile，由 ThreadLocal 内部机制保证
}
```

### ThreadPoolExecutor

```java
public class ThreadPoolExecutor {
    private volatile int ctl;  // 线程池状态 + 工作线程数（高3位状态，低29位线程数）
}
```

ctl 使用 volatile 保证多线程下状态和工作线程数的可见性。

## 易错点/踩坑

- ❌ 以为 ConcurrentHashMap 全靠 volatile 实现线程安全
- ✅ 写操作靠 synchronized + CAS，读操作靠 volatile 可见性
- ❌ 以为 volatile 的 state 就足够实现 ReentrantLock
- ✅ AQS 的核心是 CAS + volatile + CLH 队列

## 代码示例

```java
// 简化的 AQS state 操作
public class SimpleLock {
    private volatile int state = 0;

    public void lock() {
        // CAS 修改 + volatile 保证可见性
        while (!compareAndSwapState(0, 1)) {
            Thread.yield();
        }
    }

    public void unlock() {
        state = 0;  // volatile 写，对其他线程立即可见
    }

    private boolean compareAndSwapState(int expect, int update) {
        return Unsafe.getUnsafe().compareAndSwapInt(this, stateOffset, expect, update);
    }
}
```

## 关联知识点

