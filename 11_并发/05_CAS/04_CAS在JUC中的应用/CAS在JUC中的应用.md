---
title: CAS在JUC中的应用
tags:
  - Java/并发
  - 场景型
module: 05_CAS
created: 2026-04-18
---

# CAS在JUC中的应用（AtomicInteger/AtomicLong/AtomicReference/AtomicMarkableReference）

## 先说结论

CAS 是 JUC 并发工具的底层基石，广泛应用于原子类（`AtomicInteger`、`AtomicLong`、`AtomicReference`）、并发容器（`ConcurrentHashMap`）、锁（`ReentrantLock` 的 state 字段）、线程池计数等多个核心组件中。

## 深度解析

### JUC 中 CAS 的典型应用

```
JUC 框架
├── 原子类（java.util.concurrent.atomic）
│   ├── AtomicInteger    → CAS 更新 int
│   ├── AtomicLong       → CAS 更新 long
│   ├── AtomicReference  → CAS 更新对象引用
│   ├── AtomicStampedReference  → CAS + 版本号
│   └── AtomicMarkableReference → CAS + 标记位
│
├── 并发容器
│   ├── ConcurrentHashMap  → CAS 初始化数组槽位
│   ├── ConcurrentLinkedQueue → CAS 更新 head/tail
│   └── CopyOnWriteArrayList → CAS 设置数组引用
│
├── 锁框架
│   ├── ReentrantLock    → CAS 更新 state（锁状态）
│   ├── ReentrantReadWriteLock → CAS 更新 state 高16位/低16位
│   └── Semaphore        → CAS 更新 permits（许可数）
│
├── 线程池
│   ├── ThreadPoolExecutor → CAS 更新 ctl（运行状态+线程数）
│   └── FutureTask        → CAS 更新 state（任务状态）
│
└── 同步工具
    ├── CountDownLatch    → CAS 更新 state（计数器）
    └── Phaser            → CAS 更新 state（阶段+参与者）
```

### AtomicInteger 中的 CAS

```java
public class AtomicInteger {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
    private volatile int value;  // volatile 保证可见性

    // CAS 更新
    public final boolean compareAndSet(int expected, int newValue) {
        return U.compareAndSetInt(this, VALUE, expected, newValue);
    }

    // 自增（CAS + 自旋）
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }
}
```

### ReentrantLock 中的 CAS

```java
// AbstractQueuedSynchronizer
protected final boolean compareAndSetState(int expect, int update) {
    return U.compareAndSetInt(this, STATE, expect, update);
}

// 非公平锁：直接 CAS 抢锁
final void lock() {
    if (compareAndSetState(0, 1))  // CAS(state, 0, 1)
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

### ConcurrentHashMap 中的 CAS

```java
// JDK8 put 方法
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ...
    if ((tab = table) == null || (n = tab.length) == 0)
        tab = initTable();  // CAS 初始化

    // ...
    if (casTabAt(tab, i, null, new Node<>(hash, key, value)))
        break;  // CAS 空槽位直接放入
    // ...
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                     Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

## 易错点/踩坑

- ❌ 认为 AtomicInteger 的所有方法都用 CAS——`lazySet` 用的是 volatile 写（putOrderedObject），不保证立即可见
- ❌ 认为 CAS 操作一定成功——需要配合自旋循环或处理失败情况
- ✅ JUC 中的 CAS 都配合 volatile 使用，保证读操作的可见性

## 代码示例

```java
// CAS 实现线程安全栈
public class CASStack<T> {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long HEAD_OFFSET;

    static {
        try {
            HEAD_OFFSET = U.objectFieldOffset(CASStack.class, "head");
        } catch (Exception e) { throw new Error(e); }
    }

    private volatile Node<T> head;

    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        Node<T> oldHead;
        do {
            oldHead = head;
            newNode.next = oldHead;
        } while (!U.compareAndSetObject(this, HEAD_OFFSET, oldHead, newNode));
    }

    public T pop() {
        Node<T> oldHead;
        do {
            oldHead = head;
            if (oldHead == null) return null;
        } while (!U.compareAndSetObject(this, HEAD_OFFSET, oldHead, oldHead.next));
        return oldHead.value;
    }
}
```

## 关联知识点
