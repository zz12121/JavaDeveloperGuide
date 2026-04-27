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

## 易错点与踩坑

### 1. 单锁导致的"伪并发"
```java
// ArrayBlockingQueue 使用单锁
// put() 和 take() 在同一把锁下，互斥执行
// 当一个线程 put 时，另一个线程无法 take（即使队列有数据）

// 场景：生产者快，消费者慢
// 生产者：put → put → put → ...（每次都要等锁）
// 消费者：take（等待锁）→ take（等待锁）→ ...

// 结论：高吞吐场景优先选 LinkedBlockingQueue（双锁）
```

### 2. 容量必须指定，容易被忽略
```java
// ❌ 错误：以为容量会自动调整
ArrayBlockingQueue<String> q = new ArrayBlockingQueue<>(); // 编译错误！

// ✅ 必须指定容量
ArrayBlockingQueue<String> q = new ArrayBlockingQueue<>(100);

// ✅ 如果需要动态调整，只能重建队列
// 注意：重建期间队列不可用，可能丢数据
```

### 3. 公平模式下吞吐量显著下降
```java
// 公平模式：FIFO，等待最久的线程优先获取锁
// 线程A: put() → 等待锁
// 线程B: take() → 等待锁
// 线程C: put() → 等待锁
// 如果A先等，C后等 → C必须等A完成

// 非公平模式（默认）：可能插队，吞吐量更高
// 但可能导致线程饥饿（一直有新线程插入，老线程饿死）
// 高吞吐场景用非公平，低延迟/公平要求用公平
```

### 4. takeIndex/putIndex 绕回时的 CPU 空转
```java
// 环形数组索引计算
putIndex = (putIndex + 1) % capacity;  // 每次入队都要取模

// 取模操作在热点路径上，容量为2的幂次时可优化为位运算
// 如果容量不是2的幂次（如100），取模开销不可忽视

// 建议：容量设置为 2^n（如 128、256、1024）
ArrayBlockingQueue<String> q = new ArrayBlockingQueue<>(256); // 比 100 更快
```

### 5. 不能放 null，但 remove(Object) 可以删除 null 元素
```java
queue.put("a");
queue.put("b");
queue.remove("a");  // 删除后 queue = ["b"]

queue.put(null);    // 抛 NullPointerException

// 注意：如果队列中有 null（通过其他方式放入），remove(null) 会删除第一个 null
// 这是一个边界情况，生产中应该避免向队列中放入任何 null 值
```

### 6. iterator() 弱一致性的坑
```java
// ABQ 的迭代器是弱一致性的
// 可能看不到迭代开始后 put 的元素
// 在迭代过程中删除元素是安全的

// 但如果在迭代过程中有其他线程清空队列
Iterator<String> it = queue.iterator();
while (it.hasNext()) {
    String s = it.next();
    // 此时另一个线程调用 queue.clear()
    // 迭代器可能继续遍历已经被清空的位置
}
```

## 关联知识点

