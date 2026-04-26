---
title: AQS核心思想
tags:
  - Java/并发
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# AQS核心思想（状态state + FIFO队列 + CAS + Park/Unpark）

## 先说结论

AQS（AbstractQueuedSynchronizer）是 JUC 并发框架的**核心基石**，`ReentrantLock`、`CountDownLatch`、`Semaphore`、`ReentrantReadWriteLock` 等都基于它实现。核心思想：用一个 **volatile int state** 表示同步状态，用 **FIFO 双向链表** 管理等待线程，通过 **CAS** 修改 state，通过 **LockSupport.park/unpark** 阻塞/唤醒线程。

## 深度解析

### AQS 整体架构

```
AbstractQueuedSynchronizer
│
├── state (volatile int)  ← 同步状态
│   ├── ReentrantLock: 0=未锁, >0=重入次数
│   ├── CountDownLatch: 剩余计数
│   ├── Semaphore: 可用许可数
│   └── ReentrantReadWriteLock: 高16位=读锁, 低16位=写锁
│
├── FIFO CLH 队列 (双向链表)  ← 等待线程队列
│   ├── head → 指向已获取锁的节点（哨兵）
│   ├── tail → 指向最新入队的节点
│   └── Node: thread / waitStatus / prev / next
│
├── CAS (Unsafe)  ← 原子修改 state
│   └── compareAndSetState(expect, update)
│
└── LockSupport  ← 线程阻塞/唤醒
    ├── park(thread)   → 阻塞线程
    └── unpark(thread) → 唤醒线程
```

### CLH 队列结构

```
      head                                     tail
       │                                        │
    ┌──▼──┐    ┌────────┐    ┌────────┐    ┌──▼──┐
    │ null │◄───│ Node-B │◄───│ Node-C │◄───│Node-D│
    └──┬──┘    │ t=线程B │    │ t=线程C │    │t=线程D│
       │       │ ws=-1   │    │ ws=0    │    │ws=0  │
       └──────►│        │►   │        │►   │      │
              └────────┘    └────────┘    └──────┘
                    prev          prev         prev
                                   next         next

waitStatus 状态：
  SIGNAL(-1):     后继节点需要被唤醒
  CANCELLED(1):  节点已取消（超时/中断）
  PROPAGATE(-3): 共享模式下应传播唤醒
  CONDITION(-2): 在条件队列中等待
  0:             初始状态
```

### AQS 模板方法模式

```
                    AbstractQueuedSynchronizer
                    ┌─────────────────────────┐
                    │ 模板方法（final，不可重写）│
  调用方 ────────► │ acquire()                 │
                    │ release()                 │
  调用方 ────────► │ acquireShared()           │
                    │ releaseShared()           │
                    ├─────────────────────────┤
                    │ 钩子方法（子类实现）       │
                    │ tryAcquire(int)           │
                    │ tryRelease(int)           │
                    │ tryAcquireShared(int)     │
                    │ tryReleaseShared(int)     │
                    │ isHeldExclusively()       │
                    └─────────────────────────┘
                              ▲
                    ┌─────────┴──────────┐
                    │                    │
              ReentrantLock          Semaphore
              (独占模式)             (共享模式)
```

### 两种模式

```
独占模式 (Exclusive)                 共享模式 (Shared)
ReentrantLock                        CountDownLatch
WriteLock                            Semaphore
                                    ReadLock

acquire()                           acquireShared()
├── tryAcquire(arg) ──→ 子类实现     ├── tryAcquireShared(arg) ──→ 子类实现
├── 失败 → addWaiter(EXCLUSIVE)      ├── 失败 → addWaiter(SHARED)
└── acquireQueued() 自旋+park        └── doAcquireShared() 自旋+park

release()                           releaseShared()
├── tryRelease(arg) ──→ 子类实现     ├── tryReleaseShared(arg) ──→ 子类实现
└── unpark 后继节点                  └── unpark 后继节点（可能传播）
```

## 易错点/踩坑

- ❌ 认为 AQS 公平锁保证严格 FIFO——公平只是"先来先得的概率更高"
- ❌ 认为 AQS 的 state 只能表示锁——state 是通用的 int，语义由子类定义
- ✅ AQS 是模板方法模式的经典实现

## 代码示例

```java
// 自定义互斥锁（基于 AQS）
public class SimpleMutex {
    private final Sync sync = new Sync();

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            return compareAndSetState(0, 1); // CAS(state, 0, 1)
        }

        @Override
        protected boolean tryRelease(int arg) {
            setState(0); // 直接释放（独占模式下不需要 CAS）
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
    }

    public void lock() { sync.acquire(1); }
    public void unlock() { sync.release(1); }
}
```

## 关联知识点