---
title: LinkedTransferQueue
tags:
  - Java/并发
  - 问答
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedTransferQueue

## Q1：transfer、tryTransfer、put 有什么区别？

**A**：

| 方法 | 有消费者等待 | 无消费者等待 |
|------|------------|-------------|
| `transfer(e)` | 直接传递 | **阻塞**等待消费者 |
| `tryTransfer(e)` | 直接传递 | **返回 false** |
| `put(e)` | 直接传递 | **入队**等待消费者 |

transfer 要求消息必须被"接收"，put 只是放入队列。

---

## Q2：LinkedTransferQueue 和 SynchronousQueue 有什么区别？

**A**：

- **SynchronousQueue**：零容量，不存储元素，put/take 必须配对
- **LinkedTransferQueue**：无界链表，可以存储元素，等待消费者来取

LTQ 兼具 SynchronousQueue（直接传递）和 BlockingQueue（缓冲存储）的特性，更灵活。

---
```java
LinkedTransferQueue<String> queue = new LinkedTransferQueue<>();

// transfer：直接传递，无消费者则阻塞
new Thread(() -> {
    try { queue.transfer("message"); } catch (InterruptedException e) {}
}).start();

String msg = queue.take();  // 收到 "message"

// put：有消费者直接传递，无消费者则入队
queue.put("data");  // 不会阻塞

// tryTransfer：非阻塞
boolean ok = queue.tryTransfer("test");  // 有等待消费者返回 true
```

