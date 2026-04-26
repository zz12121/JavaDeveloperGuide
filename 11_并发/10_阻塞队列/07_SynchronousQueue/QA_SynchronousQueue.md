---
title: SynchronousQueue
tags:
  - Java/并发
  - 问答
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# SynchronousQueue

## Q1：SynchronousQueue 的特点是什么？

**A**：

- **不存储元素**：size 始终为 0
- **直接传递**：put 必须等 take 接收，反之亦然
- **零容量**：没有内部缓冲

put 和 take 是"握手"操作：一个线程放入，另一个线程立刻取出，数据直接在线程间传递，不经过缓冲。

---

## Q2：newCachedThreadPool 为什么用 SynchronousQueue？

**A**：CachedThreadPool 的设计目标是"有任务就创建新线程"：

- SynchronousQueue 不缓冲任务，提交后必须立刻有线程接收
- 如果有空闲线程 → 直接接收任务
- 如果没有 → 创建新线程
- 任务不会堆积在队列中

如果用有界队列，任务会堆积；如果用无界队列，可能 OOM。SynchronousQueue 确保了"来一个处理一个"的策略。

---

## Q3：SynchronousQueue 的公平和非公平模式有什么区别？

**A**：

- **公平模式**（TransferQueue）：等待线程按 FIFO 顺序匹配
- **非公平模式**（TransferStack）：后到的线程优先匹配（LIFO 栈）

非公平模式性能更高（减少线程切换），但可能导致饥饿。默认非公平。

---

## Q4：SynchronousQueue 的 transfer 是怎么实现匹配的？

**A**：核心方法 `transfer()` 实现线程间直接数据交换：

```
put 线程调用 transfer(data, true):
  1. 检查队列是否有 take 等待节点（isData=false）
  2. 有 → CAS 交换数据，unpark 唤醒 take 线程，返回
  3. 无 → 创建 QNode(data, true) 入队，park 等待
  4. 被唤醒后返回（数据已被取走）

take 线程调用 transfer(null, false):
  1. 检查队列是否有 put 等待节点（isData=true）
  2. 有 → CAS 获取数据，unpark 唤醒 put 线程，返回数据
  3. 无 → 创建 QNode(null, false) 入队，park 等待
  4. 被唤醒后返回（数据已被 put 线程填充）
```

**匹配原则**：互补类型（put 找 take，take 找 put）通过 CAS 交换数据，实现零拷贝直接传递。



> **代码示例：SynchronousQueue 的直接传递特性**

```java
SynchronousQueue<String> queue = new SynchronousQueue<>();

// 生产者：put 阻塞直到有消费者 take
new Thread(() -> {
    try {
        queue.put("hello"); // 阻塞，直到消费者取走
        System.out.println("消息已传递");
    } catch (InterruptedException e) {}
}).start();

// 消费者：take 阻塞直到有生产者 put
new Thread(() -> {
    try {
        String msg = queue.take(); // 直接从生产者线程获取，无缓冲
        System.out.println("收到: " + msg); // 输出: 收到: hello
    } catch (InterruptedException e) {}
}).start();

// queue.size() 始终为 0，因为没有内部缓冲
```

