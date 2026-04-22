---
title: Queue 接口
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# Queue 接口（offer / poll / peek）

## Queue 接口特点

`Queue` 继承自 `Collection`，模拟**队列（FIFO，先进先出）**：

```java
public interface Queue<E> extends Collection<E> {
    boolean offer(E e);   // 添加元素到队尾（推荐）
    E poll();             // 获取并移除队头（队列为空返回 null）
    E peek();             // 获取但不移除队头（队列为空返回 null）

    // 以下方法来自 Collection，但语义不同：
    boolean add(E e);     // 添加元素（失败抛异常）
    E remove();           // 获取并移除队头（空队列抛异常）
    E element();          // 获取队头（空队列抛异常）
}
```

## 两套 API 的区别

| 操作 | 不抛异常（推荐） | 抛异常 |
|------|----------------|--------|
| 添加 | `offer(e)` | `add(e)` |
| 获取并移除 | `poll()` | `remove()` |
| 获取不移除 | `peek()` | `element()` |

> **推荐使用 `offer` / `poll` / `peek`**：失败时返回 null 而不是抛异常，更适合实际业务场景。

## 主要实现类

| 实现类 | 底层 | 特点 |
|--------|------|------|
| LinkedList | 双向链表 | 同时实现 List 和 Deque |
| PriorityQueue | 最小堆 | 优先队列，非 FIFO |
| ArrayDeque | 循环数组 | 高性能双端队列 |

## 使用示例

### LinkedList 作为 Queue

```java
Queue<String> queue = new LinkedList<>();
queue.offer("a");  // 队尾添加
queue.offer("b");
queue.offer("c");

queue.poll();  // 返回 "a"（队头出队）
queue.poll();  // 返回 "b"
queue.peek();  // 返回 "c"（查看队头，不移除）
queue.poll();  // 返回 "c"
queue.poll();  // 返回 null（空队列）
```

### 队列的典型应用

```java
// BFS（广度优先搜索）
Queue<TreeNode> queue = new LinkedList<>();
queue.offer(root);
while (!queue.isEmpty()) {
    TreeNode node = queue.poll();
    // 处理当前节点
    if (node.left != null) queue.offer(node.left);
    if (node.right != null) queue.offer(node.right);
}

// 生产者-消费者模式
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();
queue.offer(new Task());  // 生产
Task task = queue.poll(); // 消费
```

## PriorityQueue（优先队列）

```java
// 默认最小堆
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1); pq.offer(2);
pq.poll();  // 1（最小元素优先）

// 最大堆
Queue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());
```

PriorityQueue 不是 FIFO，而是按优先级出队。

## 关联知识点
