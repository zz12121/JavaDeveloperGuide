---
title: AQS模板方法
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# AQS模板方法（acquire/release/acquireShared/releaseShared）

## 先说结论

AQS 使用**模板方法模式**：`acquire/release/acquireShared/releaseShared` 是 final 方法，定义了获取/释放锁的完整流程（try → 入队 → 自旋/park → 唤醒），不可被子类重写。子类只需实现 `tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared` 这四个钩子方法。

## 深度解析

### 模板方法（final，不可重写）

```java
// 独占模式
public final void acquire(int arg);          // 获取锁
public final void acquireInterruptibly(int arg); // 可中断获取
public final boolean tryAcquireNanos(int arg, long nanos); // 超时获取
public final boolean release(int arg);       // 释放锁

// 共享模式
public final void acquireShared(int arg);    // 共享获取
public final void acquireSharedInterruptibly(int arg);
public final boolean tryAcquireSharedNanos(int arg, long nanos);
public final boolean releaseShared(int arg); // 共享释放
```

### 钩子方法（protected，子类实现）

```java
// 独占
protected boolean tryAcquire(int arg);              // 尝试获取
protected boolean tryRelease(int arg);              // 尝试释放
protected boolean isHeldExclusively();              // 是否独占持有

// 共享
protected int tryAcquireShared(int arg);            // 尝试共享获取（>=0成功）
protected boolean tryReleaseShared(int arg);        // 尝试共享释放
```

### acquire() 完整流程

```java
public final void acquire(int arg) {
    // 1. 尝试获取（子类实现）
    if (!tryAcquire(arg) &&
        // 2. 失败 → 入队
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 3. 中断标记
        selfInterrupt();
}

// addWaiter：将当前线程封装为 Node，CAS 尾插法入队
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { // CAS tail
            pred.next = node;
            return node;
        }
    }
    enq(node); // CAS 失败 → 自旋入队
    return node;
}

// acquireQueued：自旋 + park
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {  // 自旋
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 成功 → 设置为 head
                p.next = null;
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

### release() 完整流程

```java
public final boolean release(int arg) {
    // 1. 尝试释放（子类实现）
    if (tryRelease(arg)) {
        Node h = head;
        // 2. head 不为 null 且 waitStatus != 0
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 3. unpark 后继节点
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0) compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从 tail 向前找到最前面的非取消节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒
}
```

### 流程图

```
acquire(arg)                         release(arg)
    │                                    │
    ▼                                    ▼
tryAcquire(arg)                     tryRelease(arg)
    │                                    │
  成功 → 返回                         成功 → unpark后继
    │                                    │
  失败 ↓                                失败 → 返回false
addWaiter(EXCLUSIVE)
    │
入队 → acquireQueued(node)
    │
    ├── 前驱==head → tryAcquire
    │   ├── 成功 → setHead → 返回
    │   └── 失败 ↓
    └── shouldParkAfterFailedAcquire
        ├── 第一次：设置前驱 ws=SIGNAL
        └── 第二次：park → 等待 unpark
                              ↑
                      release unpark 后继
```

## 易错点/踩坑

- ❌ 子类重写 acquire/release——这些是 final 方法
- ❌ tryAcquire 不需要线程安全——已持有时用 setState，首次用 CAS
- ✅ acquireQueued 中的自旋不是忙等待——会 park 阻塞

## 代码示例

```java
// 自定义不可重入互斥锁（模板方法模式）
public class NonReentrantLock implements Lock {
    private final Sync sync = new Sync();

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            // CAS 获取，不检查重入
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false; // 不支持重入，直接失败
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1 && getExclusiveOwnerThread() == Thread.currentThread();
        }

        Condition newCondition() { return new ConditionObject(); }
    }

    public void lock() { sync.acquire(1); }
    public void unlock() { sync.release(1); }
    public Condition newCondition() { return sync.newCondition(); }
    // ... 其他 Lock 接口方法
}
```

## 关联知识点
