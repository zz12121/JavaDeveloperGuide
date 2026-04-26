---
title: AQS状态管理
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# AQS状态管理（getState/setState/compareAndSetState）

## 先说结论

AQS 用一个 `volatile int state` 字段管理同步状态，提供 `getState()`、`setState()`、`compareAndSetState()` 三个方法操作它。state 的语义由子类定义——ReentrantLock 中表示锁重入次数，Semaphore 中表示许可数，CountDownLatch 中表示剩余计数。

## 深度解析

### 三个核心方法

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;

    // 获取当前状态
    protected final int getState() {
        return state;
    }

    // 设置状态（非原子，已持有锁时使用）
    protected final void setState(int newState) {
        state = newState;
    }

    // CAS 更新状态（原子操作）
    protected final boolean compareAndSetState(int expect, int update) {
        return U.compareAndSetInt(this, STATE, expect, update);
    }
}
```

### state 在不同实现中的语义

```
ReentrantLock:
  state = 0       → 未锁定
  state = n (n>0) → 已锁定，重入 n 次

ReentrantReadWriteLock:
  state 高 16 位 → 读锁持有次数
  state 低 16 位 → 写锁重入次数
  state = 0x00020003 → 2 个读锁 + 3 次写锁重入

Semaphore:
  state > 0       → 还有 state 个许可可用
  state = 0       → 没有许可，获取需等待

CountDownLatch:
  state = n       → 还需要 n 次 countDown
  state = 0       → 门闩打开，所有等待线程放行

FutureTask:
  state = NEW          → 任务未完成
  state = COMPLETING   → 正在设置结果
  state = NORMAL       → 正常完成
  state = EXCEPTIONAL  → 异常完成
  state = CANCELLED    → 已取消
```

### CAS state 的原子性保证

```java
// ReentrantLock.NonfairSync.tryAcquire
protected final boolean tryAcquire(int acquires) {
    int c = getState();
    if (c == 0) {
        // CAS(state, 0, 1)：无竞争时获取锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 重入：已持有锁，直接 setState（不需要 CAS，因为当前线程独占）
        setState(c + acquires);
        return true;
    }
    return false;
}

// ReentrantLock.Sync.tryRelease
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 当前线程不是持有者 → 非法监控状态异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c); // 重入次数减 1，或清零
    return free;
}
```

### ReentrantReadWriteLock 的 state 拆分

```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);    // 65536
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1; // 65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1; // 0xFFFF

// 读锁计数 = state >>> 16
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
// 写锁计数 = state & 0xFFFF
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

```
state = 0x00030002 (二进制: 0000 0000 0000 0011 | 0000 0000 0000 0010)
                                        高16位=3       低16位=2

读锁：3 个线程持有读锁
写锁：2 次重入
```

## 易错点/踩坑

- ❌ 直接 setState 代替 compareAndSetState——未持有时必须用 CAS
- ❌ ReadWriteLock 读锁上限是 Integer.MAX_VALUE——实际是 65535（高16位）
- ✅ 重入时用 setState（当前线程独占，无需 CAS），首次获取用 CAS

## 代码示例

```java
// 自定义同步器：最多允许 N 个线程同时访问
public class NSharedLock {
    private final Sync sync;

    public NSharedLock(int max) {
        sync = new Sync(max);
    }

    private static class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) { setState(count); }

        @Override
        protected int tryAcquireShared(int arg) {
            // state > 0 时可以获取，state 减 1
            while (true) {
                int s = getState();
                if (s <= 0) return -1;
                int next = s - 1;
                if (compareAndSetState(s, next)) return 1;
            }
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            // 释放：state 加 1
            while (true) {
                int s = getState();
                int next = s + 1;
                if (compareAndSetState(s, next)) return true;
            }
        }
    }

    public void lock() { sync.acquireShared(1); }
    public void unlock() { sync.releaseShared(1); }
}
```

## 关联知识点
