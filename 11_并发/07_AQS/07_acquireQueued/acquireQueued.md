---
title: acquireQueued
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# acquireQueued（队列中的线程自旋等待，获取成功修改节点状态）

## 先说结论

`acquireQueued(node, arg)` 是 AQS 获取锁的核心流程：已入队的节点在**自旋**中不断检查前驱是否为 head，是则 tryAcquire 尝试获取锁，成功则设置为 head；否则通过 `shouldParkAfterFailedAcquire` 判断是否需要 park 阻塞。

## 深度解析

### 源码解析

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {   // ⭐ 自旋
            // 获取前驱节点
            final Node p = node.predecessor();
            // 前驱是 head → 有资格尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 获取成功 → 设置自己为 head（原 head 出队）
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; // 返回是否被中断过
            }
            // 获取失败 → 判断是否应该 park
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node); // 异常时取消
    }
}
```

### setHead

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;  // 清除线程引用（帮助 GC）
    node.prev = null;
}
```

设置新 head 后，head.thread = null，head.prev = null，但 head.next 仍指向下一个等待节点。

### parkAndCheckInterrupt

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);    // ⭐ 阻塞当前线程
    return Thread.interrupted(); // 被唤醒后检查中断标志
}
```

线程 park 后等待 unpark 唤醒，唤醒后检查是否在等待期间被中断过。

### 完整流程图

```
acquireQueued(node)
│
├── for (;;) 自旋
│   │
│   ├── 前驱 == head?
│   │   ├── YES → tryAcquire()
│   │   │   ├── 成功 → setHead(node) → 返回
│   │   │   │         (node 变为新 head, 原head出队)
│   │   │   └── 失败 ↓
│   │   │
│   │   └── NO ↓
│   │
│   └── shouldParkAfterFailedAcquire(prev, node)
│       │
│       ├── prev.ws == SIGNAL → 直接 park
│       ├── prev.ws > 0 (CANCELLED) → 跳过取消节点, 找到有效前驱
│       └── prev.ws == 0/PROPAGATE → CAS 设置 prev.ws = SIGNAL
│       │
│       └── 下一次循环: parkAndCheckInterrupt()
│           ├── park → 等待 unpark
│           └── 被唤醒 → 回到循环顶部 → 再次检查前驱
```

### 关键理解

```
为什么只有 head 的后继才 tryAcquire？
  → 保证 FIFO 公平性：队列中的线程按顺序获取锁
  → head 是"已获取锁"的节点（或哨兵），它的后继是下一个应该获取的

为什么要自旋？
  → head 刚释放，后继可能还没 park，自旋可以"抢"到锁而不用 park/unpark 的开销
  → 自旋一次失败后再 park，减少不必要的阻塞/唤醒

setHead 做了什么？
  → 当前节点成为新 head（thread=null, prev=null）
  → 原节点的 next=null（帮助 GC）
  → head 代表"已持有锁"的节点（或哨兵空节点）
```

## 易错点/踩坑

- ❌ 认为 acquireQueued 是忙等待（busy-wait）——只有第一次自旋是忙等，之后会 park 阻塞
- ❌ 认为 head 就是锁持有者——head 可能是哨兵（thread=null），实际持有者在第一次获取后就清除了
- ✅ acquireQueued 返回的是"等待期间是否被中断"，不是获取是否成功

## 代码示例

```java
// acquireQueued 的行为追踪
// ThreadA 获取锁 → state=1
// ThreadB 尝试获取 → tryAcquire 失败 → 入队
// ThreadC 尝试获取 → tryAcquire 失败 → 入队

// 队列状态：
// head(sentinel) ← Node(B, ws=SIGNAL) ← Node(C, ws=0)
//
// ThreadB 在 acquireQueued 中：
//   第一次循环：前驱==head → tryAcquire → 失败
//   shouldParkAfterFailedAcquire：设置 head.ws=SIGNAL
//   第二次循环：前驱==head → tryAcquire → 失败
//   parkAndCheckInterrupt → park (ThreadB 阻塞)
//
// ThreadA unlock():
//   tryRelease → state=0 → unparkSuccessor(head) → unpark(ThreadB)
//
// ThreadB 被唤醒：
//   第三次循环：前驱==head → tryAcquire → 成功
//   setHead(Node(B)) → B 成为新 head
```

## 关联知识点

