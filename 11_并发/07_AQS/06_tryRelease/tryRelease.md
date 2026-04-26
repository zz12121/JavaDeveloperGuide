---
title: tryRelease
tags:
  - Java/并发
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# tryRelease（尝试释放锁，唤醒后继节点）

## 先说结论

`tryRelease(arg)` 是 AQS 的钩子方法，由子类实现释放锁的具体逻辑。ReentrantLock 中：state 减 1，如果 state 变为 0 表示锁完全释放，返回 true，AQS 框架随后 unpark 后继节点。重入锁需要多次 release 才能完全释放。

## 深度解析

### ReentrantLock 的 tryRelease

```java
// Sync.tryRelease
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;    // state - 1
    // 当前线程不是持有者 → 非法
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        // state 归零 → 锁完全释放
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);   // 更新 state（重入时 state 减 1 但不为 0）
    return free;   // true=完全释放，false=还有重入
}
```

### 释放流程

```
ThreadA: lock() × 3 → state=3
ThreadA: unlock() → tryRelease(1)
  ├── c = 3 - 1 = 2
  ├── c != 0 → free=false
  └── setState(2) → return false → 不唤醒后继

ThreadA: unlock() → tryRelease(1)
  ├── c = 2 - 1 = 1
  ├── c != 0 → free=false
  └── setState(1) → return false → 不唤醒后继

ThreadA: unlock() → tryRelease(1)
  ├── c = 1 - 1 = 0
  ├── c == 0 → free=true → clear owner
  └── setState(0) → return true → ⭐ unpark后继
```

### release() 中的唤醒

```java
// AQS.release
public final boolean release(int arg) {
    if (tryRelease(arg)) {       // 子类实现
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);  // 唤醒 head 的后继
        return true;
    }
    return false;
}

// unparkSuccessor: 从 tail 向前找非取消节点唤醒
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0) compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) s = t;
    }
    if (s != null) LockSupport.unpark(s.thread);
}
```

### 为什么从 tail 向前找？

因为 `cancelAcquire` 时可能将 `next` 指针置空（避免 GC 引用链问题），但 `prev` 指针始终可靠。从 tail 往前遍历能找到最前面的有效等待节点。

## 易错点/踩坑

- ❌ 非 lock 持有者调用 unlock——抛 `IllegalMonitorStateException`
- ❌ tryRelease 中用 CAS 更新 state——当前线程独占时不需要 CAS，直接 setState
- ✅ 返回 true 才会触发唤醒，重入时返回 false 不唤醒

## 代码示例

```java
// 演示重入锁的多次释放
ReentrantLock lock = new ReentrantLock();
lock.lock();
lock.lock();  // state=2
lock.lock();  // state=3

lock.unlock(); // state=2, 不唤醒后继
lock.unlock(); // state=1, 不唤醒后继
lock.unlock(); // state=0, ⭐ 唤醒后继

// lock.unlock(); // ⚠️ 多释放一次 → IllegalMonitorStateException
```

## 关联知识点