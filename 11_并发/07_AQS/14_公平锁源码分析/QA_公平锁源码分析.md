---
title: 公平锁源码分析
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# 公平锁源码分析（hasQueuedPredecessors检查CLH队列前驱，有则让步保证FIFO）

## Q1：公平锁的 tryAcquire 和非公平锁有什么区别？

**A**：

两者的**唯一源码级差异**在于 `state == 0` 时的判断条件：

```java
// 非公平锁 NonfairSync → nonfairTryAcquire()
if (c == 0) {
    if (compareAndSetState(0, acquires)) {  // 直接 CAS 抢锁
        setExclusiveOwnerThread(current);
        return true;
    }
}

// 公平锁 FairSync → tryAcquire()
if (c == 0) {
    if (!hasQueuedPredecessors() &&           // 多了一步：检查队列前驱
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

| 对比维度 | 非公平锁 | 公平锁 |
|---------|---------|--------|
| state==0 时行为 | 直接 `CAS(0,1)` 抢锁 | 先调 `hasQueuedPredecessors()` |
| 队列有等待者时 | 仍尝试 CAS 抢锁（可能插队） | 返回 false，放弃本次 CAS |
| 重入逻辑 | 相同（`current == owner`） | 相同（`current == owner`） |

> 核心要点：公平锁通过 `hasQueuedPredecessors()` 增加了一层"谦让"检查——发现有人在排队就让步，不参与 CAS 竞争。

---

## Q2：hasQueuedPredecessors() 方法的实现逻辑？

**A**：

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;                     // ① 读尾指针
    Node h = head;                     // ② 读头指针
    Node s;
    return h != t &&                   // ③ 队列不为空
        ((s = h.next) == null ||       // ④ head.next 为空（有线程正在入队）
         s.thread != Thread.currentThread()); // ⑤ 首等待者不是当前线程
}
```

**逐条件解析**：

- **`h != t`**：队列是否非空。`head == tail` 说明没有等待者，直接返回 `false`（无需排队）
- **`(s = h.next) == null`**：`head` 的后继为空。此时有线程正在执行 `addWaiter`，CAS tail 成功但还没执行 `t.next = node`，属于**中间状态**，应视为有前驱
- **`s.thread != currentThread()`**：队首等待节点的线程不是当前线程。如果相等说明是**重入场景**——持有锁的线程再次 `lock()`，head.next 指向的就是自己之前的节点

**返回值含义**：
- 返回 `true` → 有前驱等待者 → 公平锁**放弃本次 CAS**，进入排队
- 返回 `false` → 无前驱 → 公平锁**可以尝试 CAS**

---

## Q3：为什么默认使用非公平锁？

**A**：

**性能原因**：非公平锁减少了线程上下文切换的开销。

```
场景分析：T1 持锁，T2 在队列中 park 等待

T1.unlock() 时刻，发生了什么：

非公平锁：
  T3 新到达 → CAS(0,1) 成功 → 直接获取锁 ✅
  零次 park/unpark 切换

公平锁：
  T3 新到达 → hasQueuedPredecessors()=true → 入队 → park → 等待
  T2 被 unpark 唤醒 → tryAcquire → 成功
  T3 再次被 unpark → tryAcquire → 成功
  2 次 park/unpark 切换（T2 + T3）
```

**具体原因**：

1. **park/unpark 开销大**：线程挂起和唤醒涉及操作系统内核态切换（系统调用、线程调度），比一次失败的 CAS 昂贵得多
2. **唤醒有延迟**：T1 调用 `unlock` → `unparkSuccessor` → T2 从 `park` 中恢复 → 回到用户态 → 执行 `tryAcquire`，这个过程有时间窗口，新线程可以趁此间隙直接 CAS 成功
3. **减少饥饿的收益不大**：在实际业务中，线程竞争锁的时间窗口通常很短，饥饿概率很低，为了"可能的公平"牺牲确定的吞吐量不划算

> **Doug Lea 的设计哲学**：在大多数应用中，吞吐量比绝对公平更重要。`synchronized` 和 `ReentrantLock` 默认都选择非公平锁。

