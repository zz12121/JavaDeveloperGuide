---
title: PriorityBlockingQueue
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# PriorityBlockingQueue

## 核心结论

PriorityBlockingQueue 是**无界优先级阻塞队列**，基于数组 + 二叉小顶堆实现。元素按自然排序或 Comparator 排序，take 返回优先级最高的元素。

## 深度解析

### 数据结构

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E> {
    private transient Object[] queue; // 二叉堆（数组）
    private final ReentrantLock lock;
    private final Condition notEmpty;
    private Comparator<? super E> comparator; // 可选比较器
    private int size;
    // 无界：可自动扩容
}
```

### 二叉堆特性

```
        1
       / \
      3   2
     / \ / \
    5  4 7  6

数组存储：[1, 3, 2, 5, 4, 7, 6]
- queue[0] 是最小元素（小顶堆）
- queue[i] 的左子: queue[2i+1]，右子: queue[2i+2]
- queue[i] 的父: queue[(i-1)/2]
```

### 入队（offer/put）

```java
// 入队：O(log n)
public boolean offer(E e) {
    lock.lock();
    try {
        int n = size;
        if (n >= queue.length)
            grow(); // 自动扩容（无界）
        size = n + 1;
        if (n == 0)
            queue[0] = e;
        else
            siftUp(n, e); // 上浮调整堆
        notEmpty.signal();
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 出队（take/poll）

```java
// 出队：O(log n)
public E take() throws InterruptedException {
    lock.lockInterruptibly();
    try {
        while (size == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    E result = (E) queue[0]; // 取堆顶（最小/优先级最高）
    E x = (E) queue[--size]; // 取最后一个元素
    queue[size] = null;
    if (size != 0)
        siftDown(0, x); // 下沉调整堆
    return result;
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 有界性 | 无界（自动扩容） |
| 排序 | 自然排序或 Comparator |
| 阻塞 | take 阻塞，put 永不阻塞（无界） |
| 复杂度 | offer/take O(log n)，peek O(1) |
| 相等元素 | 不保证 FIFO 顺序 |

### 扩容机制

PriorityBlockingQueue 底层数组初始容量为 11，**自动扩容**策略如下：

```java
// grow() 扩容源码
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // 新容量 = 旧容量 + (旧容量 < 64 ? 旧容量 + 2 : 旧容量 >> 1)
    // 即：小容量时翻倍+2，大容量时1.5倍
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    if (newCapacity < 0) // overflow
        newCapacity = Integer.MAX_VALUE;
    if (newCapacity < minCapacity)
        newCapacity = minCapacity;
    queue = Arrays.copyOf(queue, newCapacity); // 数组拷贝
}
```

| 当前容量 | 扩容后容量 | 扩容倍数 |
|---------|-----------|---------|
| 11 | 24 | ~2.2x |
| 24 | 50 | ~2.1x |
| 50 | 101 | ~2x |
| 100 | 150 | 1.5x |
| 1000 | 1500 | 1.5x |

**关键点**：
- 小容量（<64）时近乎翻倍，减少频繁扩容
- 大容量时 1.5 倍增长，平衡内存和扩容频率
- 扩容需要 **数组拷贝**（`Arrays.copyOf`），O(n) 开销，大容量时谨慎

### 注意事项

- **无界**：不限制大小，注意 OOM 风险
- **put 不阻塞**：因为无界，put 永远成功
- **元素必须可比较**：实现 Comparable 或提供 Comparator
- **迭代器不保证顺序**：迭代顺序与优先级无关
- **扩容开销**：大容量时扩容需要数组拷贝，避免频繁扩容

## 关联知识点
