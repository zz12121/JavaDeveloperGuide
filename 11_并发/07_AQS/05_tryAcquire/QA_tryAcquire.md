---
title: tryAcquire
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# tryAcquire（尝试获取锁，成功返回true，失败入队阻塞）

## Q1：tryAcquire 在公平锁和非公平锁中有什么区别？

**A**：

| 维度 | 非公平 tryAcquire | 公平 tryAcquire |
|------|-----------------|----------------|
| 队列检查 | ❌ 直接 CAS 抢锁 | ✅ `hasQueuedPredecessors()` 检查 |
| 插队可能 | ✅ 新线程可能抢到锁 | ❌ 有等待者必须排队 |
| 吞吐量 | 更高 | 更低 |
| 公平性 | 低 | 高 |

```java
// 非公平：state==0 就 CAS
if (compareAndSetState(0, acquires)) return true;

// 公平：state==0 且 队列无等待者 才 CAS
if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) return true;
```

---

## Q2：tryAcquire 中重入是怎么处理的？

**A**：如果当前线程已经是锁持有者，直接 state + 1（不需要 CAS）：

```java
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) throw new Error("Maximum lock count exceeded");
    setState(nextc); // 直接设置，当前线程独占安全
    return true;
}
```

这是可重入的保证：同一线程多次 lock 不会死锁。

---

## Q3：tryAcquire 失败后会怎样？

**A**：AQS 框架接手：

1. `addWaiter(EXCLUSIVE)` — 当前线程封装为 Node，尾插法入队
2. `acquireQueued(node, arg)` — 自旋检查前驱是否为 head
3. 是 head → 再次 tryAcquire
4. 不是 head → `park` 阻塞等待 `unpark` 唤醒

