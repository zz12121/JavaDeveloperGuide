---
title: newCondition
tags:
  - Java/并发
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# newCondition

## 核心结论

`newCondition()` 创建独立的 Condition 对象，每个 Condition 维护一个**独立的等待队列**。相比 Object 的 wait/notify 只有一个等待队列，Condition 支持精确控制不同等待条件。

## 深度解析

### API 对比

| Object 监视器 | Condition | 说明 |
|--------------|-----------|------|
| `wait()` | `await()` | 释放锁，进入等待 |
| `wait(timeout)` | `await(timeout)` | 超时等待 |
| `notify()` | `signal()` | 唤醒一个等待者 |
| `notifyAll()` | `signalAll()` | 唤醒全部等待者 |
| — | `awaitUninterruptibly()` | 不可中断等待 |
| — | `awaitNanos()` | 纳秒等待 |
| — | `awaitUntil(Date)` | 截止时间等待 |

### ConditionObject 内部结构

```
ReentrantLock
├── CLH 同步队列（等待获取锁）
└── Condition 1
│   └── 等待队列（firstWaiter → lastWaiter）
└── Condition 2
    └── 等待队列（firstWaiter → lastWaiter）
```

- 每个 Condition 内部维护一个**单向链表**作为等待队列
- 等待队列中的节点和 CLH 同步队列**共用 Node 类**

### await/signal 流程

```
await():
  1. 当前线程加入 Condition 等待队列（尾插）
  2. 完全释放锁（state → 0），唤醒 CLH 队列后继节点
  3. park 当前线程（阻塞等待）
  4. 被唤醒 → 转移到 CLH 同步队列
  5. 在 CLH 队列中自旋等待获取锁
  6. 获取锁后从 await() 返回

signal():
  1. 从 Condition 等待队列取第一个节点
  2. 将该节点转移到 CLH 同步队列（enq）
  3. 唤醒该线程（LockSupport.unpark）
```

### await/signal 核心原理

`Condition.await()` 的完整源码（park 流程、节点转移机制）见 **AQS 10_ConditionObject**。

核心流程：
```
await():
  ① 加入 Condition 等待队列
  ② 完全释放锁（state → 0），唤醒 CLH 队列后继
  ③ LockSupport.park(this) 阻塞
  ④ 被 signal 唤醒 → 转移到 CLH 同步队列
  ⑤ 在 CLH 中自旋竞争锁 → 返回

signal():
  ① 取 Condition 队列第一个节点
  ② 转移到 CLH 同步队列
  ③ LockSupport.unpark() 唤醒
```

> 📖 **关联阅读**：[AQS 10_ConditionObject](/11_并发/07_AQS/10_ConditionObject/ConditionObject.md) — await/signal/doSignal 完整源码分析。

### 典型场景：有界缓冲区

```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();  // 非满条件
    private final Condition notEmpty = lock.newCondition(); // 非空条件

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity)
                notFull.await();    // 满了等待 notFull
            queue.add(item);
            notEmpty.signal();      // 通知 notEmpty
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty())
                notEmpty.await();   // 空了等待 notEmpty
            T item = queue.poll();
            notFull.signal();       // 通知 notFull
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

## 关联知识点

