---
title: shouldParkAfterFailedAcquire
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# shouldParkAfterFailedAcquire（多次前驱失败后park）

## 先说结论

`shouldParkAfterFailedAcquire` 是 AQS 中决定节点是否应该 park 阻塞的方法。它检查前驱节点的 `waitStatus`：如果是 SIGNAL，说明前驱释放锁时会 unpark 自己，可以安全 park；如果是 CANCELLED，跳过前驱找到有效节点；否则将前驱设为 SIGNAL。

## 深度解析

### 源码

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 前驱已设置为 SIGNAL → 释放时会 unpark 自己 → 可以 park
        return true;

    if (ws > 0) {
        // 前驱已取消 → 跳过，找到有效前驱
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // ws == 0 或 PROPAGATE → CAS 设为 SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

### 三种情况

```
情况1：pred.waitStatus == SIGNAL (-1)
  → 前驱已承诺"释放时 unpark 我"
  → 返回 true → 可以 park
  → 下一次循环调用 parkAndCheckInterrupt()

情况2：pred.waitStatus > 0 (CANCELLED = 1)
  → 前驱已取消 → 跳过它，向前找到第一个非取消节点
  → 修改 prev/next 指针，跳过取消节点
  → 返回 false → 继续自旋（下一次循环重新检查）

情况3：pred.waitStatus == 0 或 -3 (PROPAGATE)
  → 前驱状态不正确 → CAS 设为 SIGNAL
  → 返回 false → 继续自旋（下一次循环检查时会走情况1）
```

### 配合流程

```
第一次 tryAcquire 失败:
  shouldParkAfterFailedAcquire
  → pred.ws = 0 → CAS 设置为 SIGNAL
  → return false → 继续自旋

第二次 tryAcquire 失败:
  shouldParkAfterFailedAcquire
  → pred.ws = SIGNAL
  → return true

  parkAndCheckInterrupt()
  → park! 阻塞等待 unpark

  被唤醒后:
  → 回到 for(;;) 循环顶部
  → 检查前驱 == head → tryAcquire
```

### 为什么需要两次循环才 park？

```
第一次：设置前驱 ws=SIGNAL（确保前驱知道要通知自己）
第二次：确认 SIGNAL 已设置 → 安全 park

如果直接 park 而不设置 SIGNAL：
  前驱可能在自己 park 之后才释放锁
  前驱释放时检查 ws != SIGNAL → 不会 unpark
  → 当前线程永远不会被唤醒！死锁！
```

## 易错点/踩坑

- ❌ 认为 shouldParkAfterFailedAcquire 一定会 park——第一次通常返回 false（需要先设 SIGNAL）
- ❌ 跳过 CANCELLED 节点的操作是原子的——实际上在自旋中逐步清理
- ✅ SIGNAL 的含义是"前驱释放时会通知我"，是 park 的前提条件

## 代码示例

```java
// shouldParkAfterFailedAcquire 的状态变化追踪
// 初始队列：head(ws=0) ← NodeB(ws=0)

// ThreadB 第一次 tryAcquire 失败：
// shouldParkAfterFailedAcquire(head, NodeB)
// → head.ws == 0 → CAS(head.ws, 0, SIGNAL) → head.ws = -1
// → return false

// ThreadB 第二次 tryAcquire 失败：
// shouldParkAfterFailedAcquire(head, NodeB)
// → head.ws == SIGNAL → return true
// → parkAndCheckInterrupt() → park!

// ThreadA release():
// tryRelease → state=0
// head.ws == SIGNAL → unparkSuccessor(head) → unpark(ThreadB)
// ThreadB 被唤醒 → 自旋 tryAcquire
```

## 关联知识点
