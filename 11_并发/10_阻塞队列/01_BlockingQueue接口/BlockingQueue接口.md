---
title: BlockingQueue接口
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# BlockingQueue接口

## 核心结论

BlockingQueue 是阻塞队列的顶层接口，继承 Queue 和 Collection。核心特性：**队列满时 put 阻塞，队列空时 take 阻塞**。是生产者-消费者模式的核心组件。

## 深度解析

### API 分为四组

| 方法 | 队列满 | 队列空 | 说明 |
|------|--------|--------|------|
| `add(e)` | 抛 IllegalStateException | 抛 NoSuchElementException | Collection 继承 |
| `offer(e)` | 返回 false | 返回 false | 非阻塞 |
| `put(e)` | 阻塞等待 | — | 阻塞 |
| `offer(e, time, unit)` | 等待超时返回 false | — | 超时阻塞 |
| `remove()` | — | 抛 NoSuchElementException | Collection 继承 |
| `poll()` | — | 返回 null | 非阻塞 |
| `take()` | — | 阻塞等待 | 阻塞 |
| `poll(time, unit)` | — | 等待超时返回 null | 超时阻塞 |
| `element()` | — | 抛 NoSuchElementException | 窥视不删除 |
| `peek()` | — | 返回 null | 窥视不删除 |
| `drainTo(c)` | — | — | 批量移除 |
| `remainingCapacity()` | — | — | 剩余容量 |

### 实现类

```
BlockingQueue
├── ArrayBlockingQueue    — 有界数组
├── LinkedBlockingQueue   — 链表（有界/无界）
├── LinkedBlockingDeque   — 双端阻塞队列
├── PriorityBlockingQueue — 无界优先级
├── DelayQueue            — 延迟队列
├── SynchronousQueue      — 同步队列（不存储）
└── LinkedTransferQueue   — 传输队列
```

### 核心特点

1. **线程安全**：所有实现类都是线程安全的
2. **不支持 null**：put(null) 抛 NullPointerException
3. **可选有界**：建议指定容量，避免 OOM
4. **批量操作**：drainTo() / addAll() 支持批量

### 代码示例

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// 四种入队方式
queue.add("a");                           // 满时抛异常
boolean ok = queue.offer("b");           // 满时返回 false
queue.put("c");                           // 满时阻塞等待
queue.offer("d", 3, TimeUnit.SECONDS);    // 满时等待3秒，超时返回 false

// 四种出队方式
String e1 = queue.remove();              // 空时抛异常
String e2 = queue.poll();                // 空时返回 null
String e3 = queue.take();                // 空时阻塞等待
String e4 = queue.poll(3, TimeUnit.SECONDS); // 空时等待3秒，超时返回 null

// 批量操作
List<String> batch = new ArrayList<>();
queue.drainTo(batch, 5);  // 最多移除5个到batch
```

## 易错点/踩坑

1. **add() vs put() 的区别**：add() 满时抛 `IllegalStateException`，如果用 add() 做入队，上层必须 try-catch 或提前判断；生产者-消费者场景下推荐 put() 自动阻塞。

2. **drainTo() 是非原子的**：drainTo 在加锁后批量移除，但锁的获取/释放是一次性的，不会分批加锁。如果需要原子性批量操作，用 drainTo 而不是多次 poll。

3. **contains() 和 remove(Object) 是 O(n) 的**：遍历队列查找元素，队列很大时性能差。如果需要快速查找，考虑使用带索引的数据结构。

4. **size() 不一定精确**：并发环境下 size() 在调用后可能立即变化，仅用于监控，不要用于业务判断。

5. **iterator() 是弱一致性的**：迭代时不加锁，可能看到也可能看不到迭代开始后新增的元素。不要在迭代时做结构修改。

## 关联知识点

