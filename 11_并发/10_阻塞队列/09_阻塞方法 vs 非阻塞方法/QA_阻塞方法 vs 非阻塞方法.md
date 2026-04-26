---
title: 阻塞方法 vs 非阻塞方法
tags:
  - Java/并发
  - 问答
  - 对比型
module: 10_阻塞队列
created: 2026-04-18
---

# 阻塞方法 vs 非阻塞方法

## Q1：BlockingQueue 的四组 API 有什么区别？

**A**：

| 类型 | 插入 | 移除 | 队列满/空时 |
|------|------|------|-----------|
| 抛异常 | add(e) | remove() | 抛异常 |
| 返回值 | offer(e) | poll() | 返回 false/null |
| 阻塞 | put(e) | take() | 阻塞等待 |
| 超时 | offer(e,t,u) | poll(t,u) | 超时返回 false/null |

---

## Q2：什么场景用阻塞方法，什么场景用超时方法？

**A**：

- **阻塞方法（put/take）**：生产者-消费者模式，自动平衡生产和消费速度
- **超时方法（offer/poll + timeout）**：需要避免无限等待，如定时任务、避免死锁
- **非阻塞方法（offer/poll）**：尝试性操作，不需要等待

实际开发中，**超时方法最常用**，既避免无限阻塞又保证功能完整。

---

## Q3：drainTo 有什么用？

**A**：`drainTo(collection)` 一次性将队列中所有元素批量移到目标集合。

优势：
- 单次操作批量移动，比多次 poll 效率高
- 减少锁竞争次数
- 适合批量消费场景

---
```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

queue.put("a"); queue.put("b"); queue.put("c");
// queue.put("d");  // 阻塞，队列已满

queue.offer("d");   // 返回 false，不阻塞
queue.offer("d", 2, TimeUnit.SECONDS);  // 最多等 2 秒

// drainTo：批量消费
List<String> batch = new ArrayList<>();
queue.drainTo(batch);  // 一次性移出所有元素
queue.drainTo(batch, 2);  // 最多移出 2 个
```

