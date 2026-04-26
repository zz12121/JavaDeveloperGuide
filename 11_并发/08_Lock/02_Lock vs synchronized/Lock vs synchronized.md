---
title: Lock vs synchronized
tags:
  - Java/并发
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# Lock vs synchronized（可中断/可超时/公平/非公平/多个条件变量/tryLock尝试获取）

## 先说结论

`Lock`（以 `ReentrantLock` 为代表）相比 `synchronized` 提供了更丰富的功能：可中断、可超时、可尝试获取、公平/非公平选择、多条件变量。但 `synchronized` 更简洁、JVM 优化更成熟。**没有绝对的好坏，根据场景选择**。

## 深度解析

### 功能对比

| 特性 | synchronized | Lock (ReentrantLock) |
|------|-------------|---------------------|
| 使用方式 | 隐式（代码块/方法） | 显式（lock/unlock） |
| 释放方式 | 自动（作用域结束） | 手动（必须 finally） |
| 可中断 | ❌ | ✅ lockInterruptibly() |
| 可超时 | ❌ | ✅ tryLock(timeout) |
| 非阻塞尝试 | ❌ | ✅ tryLock() |
| 公平锁 | ❌ | ✅ new ReentrantLock(true) |
| 条件变量 | 单个（wait/notify） | 多个（newCondition()） |
| 锁状态查询 | ❌ | ✅ isLocked/isHeldByCurrentThread/getQueueLength |
| 可重入 | ✅ | ✅ |
| 底层实现 | JVM 层面（对象头+Monitor） | Java API 层面（AQS+CAS） |
| 性能 | JDK6+ 差距缩小 | 高竞争时略优 |

### 代码对比

```java
// synchronized 方式
synchronized (lock) {
    // 临界区
    lock.wait();     // 释放锁等待
    lock.notifyAll(); // 唤醒所有
}

// Lock 方式
lock.lock();
try {
    // 临界区
    Condition cond = lock.newCondition();
    cond.await();     // 释放锁等待
    cond.signalAll(); // 唤醒所有
} finally {
    lock.unlock();
}
```

### 多条件变量优势

```java
// synchronized：只有一个等待队列，无法精确唤醒
// Lock：可以创建多个条件变量，精确控制
Lock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();  // 非空条件
Condition notFull = lock.newCondition();   // 非满条件

// 生产者
lock.lock();
try {
    while (full) notFull.await();     // 等待非满
    put(item);
    notEmpty.signal();                // 通知消费者
} finally { lock.unlock(); }

// 消费者
lock.lock();
try {
    while (empty) notEmpty.await();   // 等待非空
    item = take();
    notFull.signal();                 // 通知生产者
} finally { lock.unlock(); }
```

### 选择建议

```
用 synchronized：
├── 代码简洁，不需要 Lock 的额外功能
├── 竞争不激烈的场景
├── JDK6+ 自适应锁优化已经很好
└── 不需要公平锁、超时、中断等

用 Lock：
├── 需要可中断/可超时获取锁
├── 需要公平锁
├── 需要多个条件变量（如生产者-消费者）
├── 需要尝试获取锁避免死锁
├── 需要查询锁状态
└── 需要实现自定义同步器
```

## 易错点/踩坑

- ❌ 认为 Lock 一定比 synchronized 快——JDK6+ synchronized 自适应锁优化后差距很小
- ❌ Lock 忘记 finally unlock——必须手动释放
- ✅ 优先使用 synchronized，只在需要 Lock 特定功能时才用 Lock

## 代码示例

```java
// synchronized 实现的有界缓冲区（只能一个条件队列）
public class SyncBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    public SyncBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() >= capacity) wait();
        queue.add(item);
        notifyAll(); // 唤醒所有（生产者和消费者都被唤醒）
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) wait();
        T item = queue.poll();
        notifyAll(); // 唤醒所有
        return item;
    }
}

// Lock 实现的有界缓冲区（可以精确唤醒）
public class LockBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() >= capacity) notFull.await();
            queue.add(item);
            notEmpty.signal(); // 只唤醒消费者
        } finally { lock.unlock(); }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await();
            T item = queue.poll();
            notFull.signal(); // 只唤醒生产者
            return item;
        } finally { lock.unlock(); }
    }
}
```

## 关联知识点
