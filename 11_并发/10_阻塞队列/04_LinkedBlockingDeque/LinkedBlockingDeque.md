---
title: LinkedBlockingDeque
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedBlockingDeque

## 核心结论

LinkedBlockingDeque 是基于**双向链表**的双端阻塞队列，支持从两端插入和删除。使用单 ReentrantLock + 两个 Condition。

## 深度解析

### API

```java
// 头部操作
putFirst(e) / takeFirst() / offerFirst(e) / pollFirst() / peekFirst()
// 尾部操作
putLast(e) / takeLast() / offerLast(e) / pollLast() / peekLast()
// 栈操作
push(e) / pop()  // 等价于 addFirst / removeFirst
```

### 结构

```java
static final class Node<E> {
    E item;
    Node<E> prev;  // 双向
    Node<E> next;
}
// 单 ReentrantLock + notEmpty / notFull 两个 Condition
```

### vs LinkedBlockingQueue

| 维度 | LBQ | LBD |
|------|-----|-----|
| 方向 | 单端（FIFO） | 双端 |
| 链表 | 单向 | 双向 |
| 锁 | 双锁 | 单锁（双端需要统一） |
| 用途 | 队列 | 队列 + 栈 + 工作窃取 |

### 适用场景

1. **用户层工作窃取**：生产者从队列一端放任务，多个消费者从另一端取任务（类似 ForkJoinPool 的工作窃取思想）
2. **双端处理**：生产者从一头放，消费者从另一头取
3. **栈结构**：后进先出场景

> ⚠️ **注意**：ForkJoinPool 内部**不使用** LinkedBlockingDeque，而是使用自己实现的 `WorkQueue`（数组 + CAS）。LBD 更适合用户层面的工作窃取场景。

### 代码示例

```java
LinkedBlockingDeque<String> deque = new LinkedBlockingDeque<>(10);

// 双端插入
deque.addFirst("head");
deque.addLast("tail");

// 阻塞式双端取出
String first = deque.takeFirst();
String last = deque.takeLast();

// 栈操作（LIFO）
deque.push("A");  // 等价于 addFirst
String top = deque.pop();  // 等价于 removeFirst

// 工作窃取：本线程从头部取，其他线程从尾部偷
```

## 易错点/踩坑

1. **为什么 LBD 用单锁而 LBQ 用双锁？** LBD 支持双端操作，putFirst 和 putLast 可能同时操作链表的两端，如果用双锁，两把锁之间的协调非常复杂（需要锁排序避免死锁），用单锁虽然牺牲了部分并发度，但实现简单正确。LBQ 的单端操作天然适合双锁分离。

2. **作为栈使用时的并发度**：push/pop 都操作头部，在单锁下会互相阻塞。如果需要高并发栈，考虑 `ConcurrentLinkedDeque`（无锁非阻塞）。

3. **capacity 必须指定**：`new LinkedBlockingDeque()` 默认容量为 Integer.MAX_VALUE，与 LBQ 一样有 OOM 风险。

4. **ForkJoinPool 不使用 LBD**：ForkJoinPool 内部使用自己实现的 `WorkQueue`（数组 + CAS），不是 LinkedBlockingDeque。LBD 只是实现了类似的工作窃取思想，适合用户层面的双端队列场景。

## 关联知识点