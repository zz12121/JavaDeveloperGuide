---
title: ArrayBlockingQueue
tags:
  - Java/并发
  - 源码型
module: 10_阻塞队列
created: 2026-04-18
---

# ArrayBlockingQueue

## 核心结论

ArrayBlockingQueue 是**有界数组**实现的阻塞队列，FIFO 顺序。使用单个 ReentrantLock + 两个 Condition（notEmpty / notFull）控制生产和消费。

## 深度解析

### 数据结构

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E> {
    final Object[] items;        // 数组
    int takeIndex;               // 出队索引
    int putIndex;                // 入队索引
    int count;                   // 元素数量
    final ReentrantLock lock;    // 单锁
    private final Condition notEmpty;  // 非空条件
    private final Condition notFull;   // 非满条件
}
```

### 环形数组

```
items[0] [1] [2] [3] [4] [5] [6] [7]  capacity=8
putIndex=5  takeIndex=2  count=3

入队: items[putIndex] = e, putIndex = (putIndex+1) % capacity
出队: e = items[takeIndex], takeIndex = (takeIndex+1) % capacity
```

### put 源码

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)   // 队列满
            notFull.await();            // 等待 notFull
        enqueue(e);                     // 入队
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal(); // 通知 notEmpty
}
```

### take 源码

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)            // 队列空
            notEmpty.await();          // 等待 notEmpty
        return dequeue();              // 出队
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    final Object[] items = this.items;
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    notFull.signal(); // 通知 notFull
    return x;
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 有界 | 必须指定容量 |
| FIFO | 严格先进先出 |
| 锁 | 单 ReentrantLock |
| 公平 | 可选（构造参数 fair） |
| 内存 | 固定大小，无扩容 |

### vs LinkedBlockingQueue

| 维度 | ABQ | LBQ |
|------|-----|-----|
| 底层 | 数组 | 链表 |
| 容量 | 必须指定 | 可选，默认 Integer.MAX_VALUE |
| 锁 | 单锁（put/take 竞争） | 双锁（putLock/takeLock） |
| 吞吐量 | 略低 | 略高 |
| 内存 | 固定 | 链表节点开销 |

## 关联知识点

