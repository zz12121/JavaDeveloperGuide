---
title: LinkedTransferQueue
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedTransferQueue

## 核心结论

LinkedTransferQueue 是基于链表的**无界传输队列**，结合了 BlockingQueue 和 SynchronousQueue 的特性。核心方法 `transfer(e)` 阻塞直到消费者接收。全程无锁（CAS 实现），性能极高。

## 深度解析

### 核心方法

```java
// transfer：阻塞直到消费者接收
void transfer(E e) throws InterruptedException;

// tryTransfer：非阻塞，有消费者则传递，否则返回 false
boolean tryTransfer(E e);

// put：不阻塞，放入队列等待消费者
void put(E e);
```

### 三个入队方法对比

| 方法 | 有等待消费者 | 无等待消费者 |
|------|------------|-------------|
| `transfer(e)` | 直接传递 | 阻塞等待 |
| `tryTransfer(e)` | 直接传递 | 返回 false |
| `put(e)` | 直接传递 | 入队等待 |

### 无锁实现

```java
// 完全使用 CAS，无 ReentrantLock
static final class Node {
    final boolean isData;    // true=数据节点，false=请求节点
    volatile Object item;    // 数据或 null
    volatile Node next;      // 链表指针
    volatile Thread waiter;  // 等待线程
}
```

- 使用 CAS 操作链表节点的 next 和 item
- 匹配成功后将 item 设为 null，唤醒等待线程

### vs SynchronousQueue

| 维度 | SynchronousQueue | LinkedTransferQueue |
|------|-----------------|---------------------|
| 容量 | 0（不存储） | 无界（可存储） |
| 无等待者时 | 阻塞（transfer）/ 入栈等待（put） | 入队等待 |
| 锁 | 公平模式用锁 | 无锁（CAS） |
| 性能 | 中 | 高 |
| TransferQueue | 不实现 | 实现 |

### 适用场景

- 生产者-消费者（可以缓冲也可以直接传递）
- 消息传递（保证消息被消费）
- 异步任务调度

### 代码示例

```java
LinkedTransferQueue<String> queue = new LinkedTransferQueue<>();

// transfer：阻塞直到消费者接收（保证消息被消费）
queue.transfer("hello"); // 如果没有消费者，会一直阻塞

// tryTransfer：非阻塞，有消费者才传递
boolean success = queue.tryTransfer("world"); // 无消费者返回 false

// put：不阻塞，放入队列等待消费者
queue.put("data"); // 即使没有消费者也不阻塞

// 消费者
String msg = queue.take(); // 阻塞等待，优先消费 transfer 的数据
```

## 易错点/踩坑

1. **transfer() 会无限阻塞**：如果一直没有消费者调用 take/poll，transfer 线程将永久阻塞。生产环境建议用带超时的 `tryTransfer(e, timeout, unit)` 或用 put() 替代。

2. **transfer vs put 的选择**：transfer 强制"即时交付"语义——数据必须立即交给消费者；put 是"存入信箱"语义——消费者稍后来取。如果对延迟不敏感，用 put 更安全。

3. **无界队列的 OOM 风险**：put() 和 tryTransfer() 失败时数据入队，如果生产速度远大于消费速度，内存会持续增长。需要在应用层做限流。

4. **size() 遍历链表**：由于无锁实现，size() 需要遍历整个链表统计，O(n) 开销。不要在高频路径调用。

5. **与 SynchronousQueue 的关键区别**：SynchronousQueue 的 put 等价于 transfer（阻塞），而 LinkedTransferQueue 的 put 是非阻塞的。这个差异决定了它们在不同场景下的适用性。

## 关联知识点
