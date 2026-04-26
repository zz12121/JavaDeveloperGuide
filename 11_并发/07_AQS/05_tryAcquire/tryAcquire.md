---
title: tryAcquire
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# tryAcquire（尝试获取锁，成功返回true，失败入队阻塞）

## 先说结论

`tryAcquire(arg)` 是 AQS 的钩子方法，由子类实现获取锁的具体逻辑。非公平模式下直接 CAS 抢锁，公平模式下先检查队列中是否有等待线程。成功返回 true，失败由 AQS 框架负责入队阻塞。

## 深度解析

### 非公平模式 tryAcquire（ReentrantLock）

```java
// NonfairSync.tryAcquire → 实际调用 Sync.nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // state=0，尝试 CAS 抢锁（不管队列有没有等待者）
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 已持有锁的线程 → 重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false; // 获取失败
}
```

### 公平模式 tryAcquire（ReentrantLock）

```java
// FairSync.tryAcquire
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // ⭐ 公平锁：先检查队列中是否有等待者
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

// 公平锁核心检查
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    // 队列中第一个等待线程不是当前线程 → 需要排队
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### 非公平 vs 公平 tryAcquire

```
非公平 tryAcquire:                    公平 tryAcquire:
  state==0?                            state==0?
    ├── YES → CAS(state,0,1)             ├── YES → hasQueuedPredecessors()?
    │   ├── 成功 → 获取锁                  │   ├── 没有等待者 → CAS(state,0,1)
    │   └── 失败 → return false          │   │   ├── 成功 → 获取锁
    └── NO → 检查重入                     │   │   └── 失败 → return false
                                          │   └── 有等待者 → return false（排队去）
        → return false                    └── NO → 检查重入
        → return false
```

### 在 acquire 流程中的位置

```
acquire(arg)
│
├── tryAcquire(arg)  ← 这里
│   ├── true → 返回（获取成功）
│   └── false ↓
├── addWaiter() → 入队
└── acquireQueued() → 自旋/park
```

## 易错点/踩坑

- ❌ tryAcquire 返回 false 就代表锁不可用——可能只是被其他线程抢先了，需要 AQS 框架入队等待
- ❌ 公平锁的 hasQueuedPredecessors 完全公平——存在 ABA 和竞态条件，不能保证严格 FIFO
- ✅ tryAcquire 不应该阻塞——它只尝试一次，不等待

## 代码示例

```java
// 演示公平 vs 非公平锁的 tryAcquire 行为差异
public class FairnessDemo {
    public static void main(String[] args) throws Exception {
        // 非公平锁：新来的线程可能直接 CAS 抢到锁，插队
        ReentrantLock unfair = new ReentrantLock(false);
        // 公平锁：新来的线程必须检查队列，有等待者则排队
        ReentrantLock fair = new ReentrantLock(true);
    }
}
```

## 关联知识点

