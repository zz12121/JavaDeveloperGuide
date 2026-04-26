---
title: PriorityBlockingQueue
tags:
  - Java/并发
  - 问答
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# PriorityBlockingQueue

## Q1：PriorityBlockingQueue 是怎么实现的？

**A**：基于**数组 + 二叉小顶堆**：

- 入队时 siftUp（上浮），O(log n)
- 出队时 siftDown（下沉），O(log n)
- peek 直接返回堆顶，O(1)
- 自动扩容，无界队列

元素按自然排序（Comparable）或 Comparator 排序。

---

## Q2：PriorityBlockingQueue 有什么注意事项？

**A**：

1. **无界**：自动扩容，可能导致 OOM
2. **put 永不阻塞**：因为无界，put 始终成功
3. **元素必须可比较**：实现 Comparable 或提供 Comparator，否则 ClassCastException
4. **相等元素不保证顺序**：compareTo 返回 0 的元素不保持 FIFO
5. **迭代器不保证优先级顺序**：需要 poll 才能按优先级取出



> **代码示例：PriorityBlockingQueue 优先级消费**

```java
PriorityBlockingQueue<Task> queue = new PriorityBlockingQueue<>();

// 按 priority 字段排序（实现 Comparable）
queue.put(new Task("低优先级", 3));
queue.put(new Task("高优先级", 1));
queue.put(new Task("中优先级", 2));

// poll 按优先级取出（take 永不阻塞，因为无界）
Task task = queue.take(); // 取出 priority=1 的高优先级任务

record Task(String name, int priority) implements Comparable<Task> {
    public int compareTo(Task o) { return Integer.compare(this.priority, o.priority); }
}
```

