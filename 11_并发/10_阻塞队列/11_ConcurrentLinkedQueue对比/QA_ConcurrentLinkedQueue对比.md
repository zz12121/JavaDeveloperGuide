---
title: QA_ConcurrentLinkedQueue对比
tags:
  - Java/并发
  - 面试题
module: 10_阻塞队列
created: 2026-04-27
---

# QA_ConcurrentLinkedQueue对比

## 面试题

### Q1: ConcurrentLinkedQueue 和 BlockingQueue 的区别是什么？

**参考答案：**

| 维度 | CLQ | BlockingQueue |
|------|-----|---------------|
| 阻塞机制 | 无阻塞 | put/take 支持阻塞等待 |
| 锁机制 | 无锁（CAS） | 有锁（ReentrantLock） |
| null 支持 | 不支持 | 不支持 |
| size() | O(n) 弱一致 | O(1) 精确 |
| 适用场景 | 高并发读多写少 | 生产者-消费者平衡 |

---

### Q2: ConcurrentLinkedQueue 的 size() 为什么是 O(n)？

**参考答案：**

CLQ 使用无锁实现，`size` 字段不是精确维护的。需要遍历整个链表统计节点数量，复杂度 O(n)。而且由于并发修改，遍历结果可能是弱一致的（看到部分更新）。

建议用 `isEmpty()` 判断而非 `size() > 0`。

---

### Q3: ConcurrentLinkedQueue 是阻塞队列吗？

**参考答案：**

不是。CLQ 是非阻塞队列，没有 `put()` 和 `take()` 方法。如果需要阻塞语义，应该使用 `BlockingQueue` 接口的实现类（如 `LinkedBlockingQueue`）。

如果需要在 CLQ 上实现阻塞，可以用 `while` 循环 + `poll(timeout)` 模拟。

---

### Q4: ConcurrentLinkedQueue 的迭代器是线程安全的吗？

**参考答案：**

迭代器是弱一致性的。迭代过程中，其他线程对队列的修改可能看到也可能看不到。不会抛 `ConcurrentModificationException`。

如果需要"快照"语义，应该先复制到 `ArrayList` 再遍历。
