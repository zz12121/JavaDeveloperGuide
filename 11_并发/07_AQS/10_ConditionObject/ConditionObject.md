---
title: ConditionObject
tags:
  - Java/并发
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# ConditionObject（await/signal/signalAll，Object的wait/notify的Lock版本）

## 先说结论

`ConditionObject` 是 AQS 的内部类，实现了 `Condition` 接口，是 `synchronized` 的 `wait/notify/notifyAll` 的 Lock 版本。支持**多个条件队列**，每个 `Condition` 有独立的等待队列，可以实现更精确的线程唤醒。

## 深度解析

### 核心方法

```java
// Condition 接口
public interface Condition {
    void await() throws InterruptedException;           // 等待
    void awaitUninterruptibly();                         // 不可中断等待
    boolean await(long time, TimeUnit unit);             // 超时等待
    boolean awaitUntil(Date deadline);                   // 截止时间等待
    void signal();                                       // 唤醒一个
    void signalAll();                                    // 唤醒所有
}
```

### await() 核心流程

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    // 1. 将当前线程加入条件队列（尾插法）
    Node node = addConditionWaiter();
    // 2. 完全释放锁（包括重入的）
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 3. 不在同步队列中 → park
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 4. 被唤醒 → 重新获取锁（恢复到 await 前的重入次数）
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters(); // 清理条件队列
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

### signal() 核心流程

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter; // 条件队列第一个节点
    if (first != null)
        doSignal(first); // 转移到同步队列
}

private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&  // ⭐ 转移到同步队列
             (first = firstWaiter) != null);
}

// transferForSignal：将条件队列节点转移到同步队列
final boolean transferForSignal(Node node) {
    // CAS 设为 0（从条件队列移除标记）
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // CAS 入同步队列尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); // 唤醒去争锁
    return true;
}
```

### 两种队列

```
同步队列（AQS）：             条件队列（Condition）：
获取锁失败的线程等待           await() 的线程等待

head ← B ← C ← tail          firstWaiter ← D ← E ← lastWaiter
(CLF 双向链表)                (单向链表，nextWaiter)

signal():
  1. 从条件队列取出 firstWaiter(D)
  2. 将 D 转移到同步队列尾部
  3. D 在同步队列中等待获取锁（acquireQueued）
  4. 获取锁后从 await() 返回
```

### 对比 wait/notify

| 维度 | wait/notify | Condition |
|------|------------|-----------|
| 锁类型 | synchronized | Lock |
| 条件队列 | 只有一个 | 可创建多个 |
| 释放锁 | 自动 | fullyRelease |
| 精确唤醒 | ❌ notifyAll 唤醒所有 | ✅ signal 精确唤醒 |
| 超时等待 | wait(timeout) | await(timeout) |
| 中断处理 | 不能 | 可以 |

## 易错点/踩坑

- ❌ 未持有锁时调用 await/signal——抛 IllegalMonitorStateException
- ❌ 认为 signal 后线程立即执行——signal 只是将线程从条件队列移到同步队列，仍需重新获取锁
- ✅ await 会完全释放锁（包括重入次数），返回时恢复

## 代码示例

```java
// 生产者-消费者（精确唤醒）
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
Queue<String> buffer = new LinkedList<>();
int capacity = 10;

// 生产者
public void produce(String item) throws InterruptedException {
    lock.lock();
    try {
        while (buffer.size() >= capacity)
            notFull.await();     // 等待非满
        buffer.add(item);
        notEmpty.signal();       // 精确唤醒消费者
    } finally { lock.unlock(); }
}

// 消费者
public String consume() throws InterruptedException {
    lock.lock();
    try {
        while (buffer.isEmpty())
            notEmpty.await();    // 等待非空
        String item = buffer.poll();
        notFull.signal();        // 精确唤醒生产者
        return item;
    } finally { lock.unlock(); }
}
```

## 关联知识点
