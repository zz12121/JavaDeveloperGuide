---
title: shouldParkAfterFailedAcquire
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# shouldParkAfterFailedAcquire（多次前驱失败后park）

## Q1：shouldParkAfterFailedAcquire 什么时候返回 true？

**A**：当前驱节点的 `waitStatus == SIGNAL` 时返回 true，表示可以安全 park。

三种情况：
1. **ws == SIGNAL** → 返回 true → 下次循环 park
2. **ws > 0 (CANCELLED)** → 跳过前驱，找到有效节点 → 返回 false
3. **ws == 0 或 PROPAGATE** → CAS 设为 SIGNAL → 返回 false

通常需要**两次循环**才 park：第一次设置前驱 SIGNAL，第二次确认后 park。

---

## Q2：为什么要先设 SIGNAL 再 park？直接 park 不行吗？

**A**：不行。SIGNAL 是前驱释放锁时 unpark 后继的**信号**。

如果直接 park 不设 SIGNAL：
- 前驱释放时检查 head.ws，发现不是 SIGNAL
- 不会调用 unparkSuccessor
- 当前线程永远不会被唤醒 → **死锁**

流程：先设 SIGNAL（确保前驱知道要通知自己）→ 再 park（安全阻塞）。

---

## Q3：CANCELLED 节点是怎么被跳过的？

**A**：通过 `prev = pred.prev` 循环跳过所有 ws > 0 的节点，找到最前面的有效节点作为新前驱，并修正 next 指针：

```java
if (ws > 0) {
    do {
        node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
}
```

这是 AQS 队列自我清理的机制：在后续节点自旋时逐步移除 CANCELLED 节点。

