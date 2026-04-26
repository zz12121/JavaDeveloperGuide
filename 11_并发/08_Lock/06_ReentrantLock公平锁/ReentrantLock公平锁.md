---
title: ReentrantLock公平锁
tags:
  - Java/并发
  - 源码型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock公平锁

## 核心结论

公平锁通过 `new ReentrantLock(true)` 创建，核心逻辑是**获取锁前先检查 CLH 队列是否有前驱等待节点**，有则不抢锁，按 FIFO 顺序获取。

## 深度解析

### 创建方式

```java
ReentrantLock fairLock = new ReentrantLock(true);   // 公平锁
ReentrantLock unfairLock = new ReentrantLock(false); // 非公平锁（默认）
```

### FairSync.tryAcquire 源码

```java
// FairSync
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 关键区别：hasQueuedPredecessors() 检查队列
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### hasQueuedPredecessors

```java
// 检查当前线程是否是队列中的第一个等待线程
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

- 队列为空（h == t）或当前线程是 head 的后继 → 返回 false，可以抢锁
- 队列不为空且当前线程不是第一个等待者 → 返回 true，必须排队

### 公平锁获取流程

```
lock()
  → tryAcquire()
    → state == 0 ?
      → hasQueuedPredecessors() ?
        → 无前驱 → CAS 设置 state → 成功
        → 有前驱 → 返回 false → 入队等待
    → state != 0 && 当前线程持有 → 重入 state+1
  → tryAcquire 失败 → addWaiter 入队 → acquireQueued 自旋
```

### 公平锁 vs 非公平锁获取时机

| 时机 | 公平锁 | 非公平锁 |
|------|--------|---------|
| state==0，队列为空 | CAS 获取 | CAS 获取 |
| state==0，队列有等待者 | **不抢，排队** | **直接 CAS 抢** |
| state!=0，同线程 | 重入 | 重入 |
| state!=0，不同线程 | 入队等待 | 入队等待 |

### 性能影响

- **公平锁吞吐量更低**：每次获取都要检查队列，即使 state 为 0 且当前线程正好是下一个也不能立即获取（hasQueuedPredecessors 不是原子操作）
- **非公平锁吞吐量更高**：刚释放锁时，唤醒线程与 CAS 插队线程竞争，插队往往更快
- **公平锁更少饥饿**：保证先到先得

## 关联知识点
