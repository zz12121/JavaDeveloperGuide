---
id: card_63
title: AQS两种模式（独占模式ReentrantLock/共享模式CountDownLatch/Semaphore）
tags:
  - Java/并发
  - 卡片
  - 必考
  - 原理型
module: 07_AQS
created: 2026-04-18
difficulty: 中
---

# AQS两种模式

## 先说结论

AQS 支持两种同步模式：**独占模式**(Exclusive)和**共享模式**（Shared）。独占模式同一时刻只允许一个线程持有同步状态（如 ReentrantLock），共享模式允许多个线程同时持有（如 CountDownLatch、Semaphore、读锁）。两者的核心流程相似，区别在于释放后是否向后传播唤醒。

## 深度解析

### 独占模式（Exclusive）

```
acquire(arg)
├── tryAcquire(arg)          ← 子类实现
│   ├── 成功 → 返回
│   └── 失败 ↓
├── addWaiter(Node.EXCLUSIVE) ← 尾插法入队
└── acquireQueued(node, arg)  ← 自旋 + park
    ├── 前驱是 head → tryAcquire
    │   ├── 成功 → 设置为 head → 返回
    │   └── 失败 → shouldParkAfterFailedAcquire → park
    └── 被唤醒 → 再次自旋

release(arg)
├── tryRelease(arg)          ← 子类实现
│   ├── 成功 ↓
│   └── 失败 → 返回 false
├── unparkSuccessor(head)     ← unpark 后继节点
└── 后继节点被唤醒 → 自旋 tryAcquire
```

### 共享模式（Shared）

```
acquireShared(arg)
├── tryAcquireShared(arg)    ← 子类实现
│   ├── >= 0 → 成功，返回
│   └── < 0 → 失败 ↓
├── doAcquireShared(arg)      ← 入队 + 自旋
│   ├── 前驱是 head → tryAcquireShared
│   │   ├── 成功 → setHeadAndPropagate ← ⭐ 向后传播唤醒
│   │   └── 失败 → park
│   └── 被唤醒 → 再次自旋
└── 返回

releaseShared(arg)
├── tryReleaseShared(arg)    ← 子类实现
│   ├── 成功 ↓
│   └── 失败 → 返回 false
├── doReleaseShared()         ← ⭐ 可能传播唤醒
│   ├── head.ws == SIGNAL → unpark 后继
│   └── head.ws == PROPAGATE → 设置 ws 为 PROPAGATE，继续传播
└── 唤醒传播
```

### 关键区别：唤醒传播

```
独占模式释放：
  ThreadA 释放锁 → unpark(head的后继) → ThreadB 获取锁
  ↑ 只唤醒一个后继，不传播

共享模式释放：
  ThreadA releaseShared → unpark(head的后继) → ThreadB 获取
    → ThreadB setHeadAndPropagate → 检查后继
      → 如果后继也是 SHARED → 继续唤醒 → ThreadC 获取
        → ... 传播链
  ↑ 唤醒传播，允许多个线程同时获取
```

### 两种模式的代表

| 模式 | 代表 | state 语义 | Node 标记 |
|------|------|-----------|----------|
| 独占 | ReentrantLock | 锁计数 | Node.EXCLUSIVE |
| 独占 | WriteLock | 写锁计数 | Node.EXCLUSIVE |
| 共享 | CountDownLatch | 剩余计数 | Node.SHARED |
| 共享 | Semaphore | 可用许可 | Node.SHARED |
| 共享 | ReadLock | 读锁计数 | Node.SHARED |
| 两者 | ReentrantReadWriteLock | 高16位读+低16位写 | 独占/共享 |

## 易错点/踩坑

- ❌ 认为共享模式不需要队列——获取失败的线程仍需入队等待
- ❌ 认为独占和共享模式不能混用——ReentrantReadWriteLock 同时支持两种
- ✅ 释放时是否"传播唤醒"是两种模式的核心区别

## 代码示例

```java
// 独占模式：自定义互斥锁
class Mutex extends AbstractQueuedSynchronizer {
    @Override
    protected boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);
    }
    @Override
    protected boolean tryRelease(int arg) {
        setState(0);
        return true;
    }
    @Override
    protected boolean isHeldExclusively() {
        return getState() == 1;
    }
}

// 共享模式：自定义共享锁（最多 N 个线程同时持有）
class SharedLock extends AbstractQueuedSynchronizer {
    SharedLock(int max) { setState(max); }
    @Override
    protected int tryAcquireShared(int arg) {
        for (;;) {
            int s = getState();
            if (s <= 0) return -1;
            if (compareAndSetState(s, s - 1)) return 1;
        }
    }
    @Override
    protected boolean tryReleaseShared(int arg) {
        for (;;) {
            int s = getState();
            if (compareAndSetState(s, s + 1)) return true;
        }
    }
}
```

## 关联知识点