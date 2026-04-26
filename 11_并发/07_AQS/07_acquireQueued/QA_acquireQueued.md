---
title: acquireQueued
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# acquireQueued（队列中的线程自旋等待，获取成功修改节点状态）

## Q1：acquireQueued 的自旋逻辑是什么？

**A**：已入队的节点在 for(;;) 循环中：

1. **检查前驱是否为 head**：是则 `tryAcquire` 尝试获取锁
2. **成功**：`setHead(node)` 成为新 head，退出循环
3. **失败**：`shouldParkAfterFailedAcquire` → 如果前驱状态为 SIGNAL 则 `park` 阻塞
4. **被 unpark 唤醒**：回到循环顶部，再次检查前驱

不是纯粹的忙等待——失败后会 park 阻塞，不浪费 CPU。

---

## Q2：为什么只有 head 的后继才 tryAcquire？

**A**：为了 **FIFO 顺序**。head 代表"已获取锁"（或哨兵），它的直接后继是队列中等待时间最长的线程，应该最先获取锁。非 head 的后继即使 tryAcquire 也可能成功（非公平锁），但正常流程下只有 head 后继才会 tryAcquire。

---

## Q3：setHead 做了什么？为什么？

**A**：`setHead(node)` 将获取锁的节点设为新 head，并清除 thread 和 prev 引用：

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;  // 清除线程引用，帮助 GC
    node.prev = null;
}
```

head 是"已处理"的节点（thread=null），真正的锁持有者已清除。head.next 仍保留，指向下一个等待者。


