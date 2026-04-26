---
title: cancelAcquire
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# cancelAcquire（取消等待，状态标记，取消后被唤醒处理）

## 先说结论

`cancelAcquire(node)` 处理等待线程的超时或中断取消。它将节点状态设为 CANCELLED，然后调整链表指针（断开与前后节点的连接），最后唤醒后继节点避免它永久阻塞。取消节点的清理不是一次性的，而是在其他线程自旋时逐步被跳过。

## 深度解析

### 触发条件

```java
// acquireQueued 中，只有异常时才 cancel
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        for (;;) {
            // ...
        }
    } finally {
        if (failed)
            cancelAcquire(node); // 异常/超时/中断导致获取失败
    }
}
```

### cancelAcquire 源码（简化）

```java
private void cancelAcquire(Node node) {
    if (node == null) return;
    node.thread = null;  // 清除线程引用

    // 跳过已取消的前驱
    Node pred = node.prev;
    while (pred.waitStatus > 0) {
        node.prev = pred = pred.prev;
    }

    // pred 是有效前驱
    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED; // 标记为取消

    // 情况1：node 是 tail → 新 tail 设为 pred
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        // 情况2：node 不是 tail
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next); // 跳过 node
        } else {
            // 情况3：pred 是 head 或无法设置 SIGNAL → unpark 后继
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

### 三种取消场景

```
场景1：node 是 tail
  ┌──────┐    ┌──────┐    ┌──────┐
  │ pred │←──│ node │←──│ tail │
  └──┬───┘    └──┬───┘    └──────┘
     │           │ ws=CANCELLED
     └─── tail=pred, pred.next=null ──→
     node 从队列中移除

场景2：node 在中间
  ┌──────┐    ┌──────┐    ┌──────┐
  │ pred │←──│ node │←──│ next │
  └──┬───┘    └──┬───┘    └──┬───┘
     │           │            │
     └─── pred.next=next ────┘
     node 被跳过（next 指向 pred）

场景3：node 的后继可能被阻塞
  └── unparkSuccessor(node) → 唤醒后继
     后继在 shouldParkAfterFailedAcquire 中会跳过 CANCELLED 的 node
```

### 关键设计

```
1. 为什么 node.next = node？
   → 断开 next 指针，帮助 GC
   → unparkSuccessor 从 tail 向前找，不受影响

2. 为什么取消不立即清理队列？
   → 立即清理需要遍历整个队列，成本高
   → 其他线程在 shouldParkAfterFailedAcquire 中逐步跳过 CANCELLED 节点
   → 懒删除，分摊清理成本

3. 为什么要 unparkSuccessor？
   → node 的后继可能正在 park 等待
   → 如果 node 不通知后继，后继可能永远阻塞
```

## 易错点/踩坑

- ❌ 取消后立即从队列物理移除——只是标记 CANCELLED，物理移除是惰性的
- ❌ 取消的线程还会被唤醒——cancelAcquire 不会 park 取消线程
- ✅ 取消操作本身也有 CAS 失败的可能，因为并发修改队列

## 代码示例

```java
// 超时获取锁的取消
ReentrantLock lock = new ReentrantLock();
try {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        try {
            // 临界区
        } finally {
            lock.unlock();
        }
    } else {
        // 超时获取失败 → 内部 cancelAcquire
        System.out.println("获取锁超时");
    }
} catch (InterruptedException e) {
    // 中断 → 内部 cancelAcquire
    System.out.println("等待锁时被中断");
}
```

## 关联知识点
