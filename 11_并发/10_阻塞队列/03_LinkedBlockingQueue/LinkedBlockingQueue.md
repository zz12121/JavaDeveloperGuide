---
title: LinkedBlockingQueue
tags:
  - Java/并发
  - 源码型
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedBlockingQueue

## 核心结论

LinkedBlockingQueue 是基于**链表**的可选有界阻塞队列。使用**双锁**（putLock / takeLock）分离生产和消费操作，吞吐量高于 ArrayBlockingQueue。默认容量 Integer.MAX_VALUE（注意 OOM 风险）。

## 深度解析

### 数据结构

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E> {
    static class Node<E> {
        E item;
        Node<E> next;
    }
    private final int capacity;     // 容量
    private final AtomicInteger count = new AtomicInteger(0);
    transient Node<E> head;         // 队头（哨兵节点）
    private transient Node<E> last; // 队尾
    private final ReentrantLock takeLock = new ReentrantLock();  // 出队锁
    private final ReentrantLock putLock = new ReentrantLock();   // 入队锁
    private final Condition notEmpty = takeLock.newCondition();  // 非空
    private final Condition notFull = putLock.newCondition();    // 非满
}
```

### 双锁设计

```
put 操作：
  putLock.lock() → 创建 Node → 链表尾插 → count.getAndIncrement() → notEmpty.signal() → putLock.unlock()

take 操作：
  takeLock.lock() → 链表头删 → count.getAndDecrement() → notFull.signal() → takeLock.unlock()
```

put 和 take 操作在**不同锁**下进行，互不阻塞，并发度更高。

### put 源码

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity)
            notFull.await();
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal(); // 还有空间，通知其他生产者
    } finally {
        putLock.unlock();
    }
    if (c == 0) signalNotEmpty(); // 第一个元素入队，通知消费者
}
```

### 为什么需要 signalNotEmpty？

```
put 后通知消费者时，需要获取 takeLock：
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

因为 notEmpty 是 takeLock 的 Condition，signal 时必须持有 takeLock。

### vs ArrayBlockingQueue

| 维度 | ABQ | LBQ |
|------|-----|-----|
| 底层 | 数组 | 链表 |
| 锁 | 单锁 | 双锁 |
| 并发度 | 低（put/take 互斥） | 高（put/take 独立） |
| 吞吐量 | 略低 | **更高** |
| 内存 | 固定 | 链表节点额外开销 |
| 默认容量 | 必须指定 | Integer.MAX_VALUE（⚠️ OOM） |

### 注意事项

- **默认无界**：`new LinkedBlockingQueue()` 不指定容量等于 Integer.MAX_VALUE，生产速度 > 消费速度时 OOM
- **线程池使用**：`Executors.newFixedThreadPool` 使用 LBQ，可能导致 OOM

## 关联知识点

