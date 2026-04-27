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

## 易错点与踩坑

### 1. 默认无界队列导致 OOM（最高频问题）
```java
// ❌ 危险：默认容量是 Integer.MAX_VALUE
LinkedBlockingQueue<Task> queue = new LinkedBlockingQueue<>();
// 等价于 new LinkedBlockingQueue<>(Integer.MAX_VALUE)

// 场景：生产者每秒提交1000个任务，消费者每秒只能处理100个
// 1秒后队列积压：900个
// 10秒后：9000个
// 100秒后：90000个 → 内存耗尽 OOM

// ✅ 生产环境必须指定容量
LinkedBlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// ✅ 或使用有界队列替代
LinkedBlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);
```

### 2. 双锁设计导致的"原子性错觉"
```java
// put 和 take 在不同的锁下执行
// ⚠️ putLock.signalNotEmpty() 需要获取 takeLock
// ⚠️ takeLock.signalNotFull() 需要获取 putLock

// 问题：count.get() 是原子的，但"检查队列非空+读取元素"不是原子的
if (queue.size() > 0) {      // 线程A检查
    String item = queue.take(); // 线程B可能先 take 完了
}

// 建议：不要基于 count 做业务判断，只用于监控
```

### 3. signalNotEmpty() 可能唤醒错误的消费者
```java
// put 完成后调用 signalNotEmpty()
// 但 takeLock.signal() 只是唤醒一个等待在 notEmpty 上的线程
// 如果有多个消费者，可能唤醒的不是"最需要"的线程

// 这是 Condition 的正常行为，不是bug
// 但在高并发场景下可能导致负载不均
```

### 4. capacity 需要在构造时确定，无法动态调整
```java
// ❌ LinkedBlockingQueue 没有 setCapacity 方法
// 只能在构造时指定

// ✅ 如果需要动态调整，只能重建队列
LinkedBlockingQueue<Task> oldQueue = queue;
queue = new LinkedBlockingQueue<>(newCapacity);
oldQueue.drainTo(queue); // 将数据迁移到新队列
// 注意：迁移期间可能有数据丢失或重复处理
```

### 5. head 是哨兵节点，容易忽略
```java
// LinkedBlockingQueue 内部结构：
// head -> [哨兵] -> [node1] -> [node2] -> ... -> last

// 哨兵节点的 item 永远是 null
// take() 返回的是 head.next 的 item，不是 head 本身

// 这意味着 size() 实际是 count.get()，不包括哨兵节点
// 遍历时注意跳过哨兵节点（队列至少有1个Node）
```

### 6. 链表节点不会自动释放
```java
// take() 后，队列中旧的 Node 对象会被 GC
// 但如果有强引用持有 Node：
Node<T> node = queue.peek(); // 持有引用
queue.take();                 // Node 从队列移除，但 node 变量还持有引用

// ⚠️ 长时间持有 Node 引用会导致内存泄漏
// 解决：用局部变量或及时清理引用
```

## 关联知识点

