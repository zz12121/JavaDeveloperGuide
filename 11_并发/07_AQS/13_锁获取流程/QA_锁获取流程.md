---
title: 锁获取流程
tags:
  - Java/并发
  - 问答
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# 锁获取流程（tryAcquire → 失败 → addWaiter入队 → acquireQueued自旋）

## Q1：请描述 AQS 独占模式获取锁的完整流程

**A**：

```
acquire(1)
│
1. tryAcquire(1)
   ├── 成功 → 获取锁 → 返回
   └── 失败 ↓

2. addWaiter(EXCLUSIVE)
   └── 当前线程封装为 Node，CAS 尾插法入队

3. acquireQueued(node, 1)
   ├── 自旋：前驱 == head → tryAcquire
   │   ├── 成功 → setHead(node) → 返回
   │   └── 失败 → shouldParkAfterFailedAcquire
   │       ├── 第一次：设前驱 ws=SIGNAL → 继续自旋
   │       └── 第二次：ws 已是 SIGNAL → park 阻塞
   └── 被 unpark 唤醒 → 回到自旋 → 再次 tryAcquire
```

---

## Q2：线程被 unpark 唤醒后直接获取锁吗？

**A**：不是。被唤醒后回到 `acquireQueued` 的 for 循环顶部，检查前驱是否为 head，是则 tryAcquire。如果在此期间有其他线程抢先获取锁，会再次 park。

```
unpark → 回到循环顶部 → 前驱==head? → tryAcquire → 成功才获取
                                                    → 失败再 park
```

---

## Q3：addWaiter 入队失败怎么办？

**A**：`addWaiter` 的快速路径是 CAS 设置 tail，如果失败则调用 `enq(node)` 自旋入队：

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 队列未初始化 → 创建空哨兵 head
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

自旋保证最终一定能入队成功。
