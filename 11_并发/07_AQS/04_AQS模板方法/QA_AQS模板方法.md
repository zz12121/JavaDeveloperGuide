---
title: AQS模板方法
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# AQS模板方法（acquire/release/acquireShared/releaseShared）

## Q1：AQS 的模板方法模式是怎么设计的？

**A**：

**模板方法（final）**：定义完整流程，不可重写
- `acquire(arg)` → tryAcquire → addWaiter → acquireQueued
- `release(arg)` → tryRelease → unparkSuccessor

**钩子方法（protected）**：由子类实现，定义具体的获取/释放逻辑
- `tryAcquire(arg)` / `tryRelease(arg)`
- `tryAcquireShared(arg)` / `tryReleaseShared(arg)`

AQS 只负责排队、park/unpark 的通用逻辑，具体的 state 语义和获取/释放规则由子类定义。

---

## Q2：acquire() 的完整流程是什么？

**A**：

```
acquire(arg)
│
├── 1. tryAcquire(arg)         ← 子类实现
│   ├── 成功 → 返回（获取锁成功）
│   └── 失败 ↓
│
├── 2. addWaiter(EXCLUSIVE)    ← 当前线程封装为 Node，尾插法入队
│
├── 3. acquireQueued(node, arg)
│   ├── 自旋检查：前驱是否是 head？
│   │   ├── 是 → tryAcquire
│   │   │   ├── 成功 → setHead(node) → 返回
│   │   │   └── 失败 → 继续
│   │   └── 不是 → shouldParkAfterFailedAcquire → park
│   └── 被唤醒 → 再次自旋
│
└── 4. 如果被中断过 → selfInterrupt()
```

---

## Q3：release() 是怎么唤醒等待线程的？

**A**：

```java
release(arg) {
    if (tryRelease(arg)) {          // 子类：释放锁
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);     // unpark head 的后继节点
    }
}

unparkSuccessor(node) {
    // 从 tail 向前找到最前面非取消的节点
    // 调用 LockSupport.unpark(thread) 唤醒
}
```

为什么从 tail 向前找？因为 next 指针可能在 cancelAcquire 时被置空，但 prev 指针始终可靠。

