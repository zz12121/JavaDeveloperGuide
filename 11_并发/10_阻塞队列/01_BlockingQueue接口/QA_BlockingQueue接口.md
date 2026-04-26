---
title: BlockingQueue接口
tags:
  - Java/并发
  - 问答
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# BlockingQueue接口

## Q1：BlockingQueue 的核心 API 有哪些？

**A**：

| 方法 | 队列满 | 队列空 | 类型 |
|------|--------|--------|------|
| `put(e)` | 阻塞 | — | 阻塞 |
| `take()` | — | 阻塞 | 阻塞 |
| `offer(e)` | 返回 false | 返回 false | 非阻塞 |
| `poll()` | — | 返回 null | 非阻塞 |
| `offer(e, time, unit)` | 超时返回 false | — | 超时 |
| `poll(time, unit)` | — | 超时返回 null | 超时 |
| `add(e)` | 抛异常 | — | 异常 |
| `remove()` | — | 抛异常 | 异常 |

---

## Q2：BlockingQueue 有哪些常用实现类？

**A**：

- **ArrayBlockingQueue**：有界数组，FIFO
- **LinkedBlockingQueue**：链表，默认无界（Integer.MAX_VALUE）
- **LinkedBlockingDeque**：双端阻塞队列
- **PriorityBlockingQueue**：无界优先级队列
- **DelayQueue**：延迟队列，元素过期才能出队
- **SynchronousQueue**：不存储元素，直接传递
- **LinkedTransferQueue**：transfer 阻塞直到消费者接收

---

## Q3：BlockingQueue 为什么不支持 null？

**A**：因为 null 被 poll/take/peek 用作"队列空"的返回值。如果允许 null，就无法区分"取到了 null 元素"和"队列空"。

---
```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

queue.put("a");          // 阻塞放入（队列满时等待）
String e = queue.take();  // 阻塞取出（队列空时等待）
queue.offer("b");        // 非阻塞放入，满返回 false
String p = queue.poll();  // 非阻塞取出，空返回 null
queue.add("c");          // 队列满抛 IllegalStateException

// 超时版本
queue.offer("d", 3, TimeUnit.SECONDS);  // 最多等 3 秒
queue.poll(3, TimeUnit.SECONDS);         // 最多等 3 秒
```


