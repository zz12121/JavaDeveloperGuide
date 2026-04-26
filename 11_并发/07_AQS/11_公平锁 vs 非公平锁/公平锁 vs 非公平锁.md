---
title: 公平锁 vs 非公平锁
tags:
  - Java/并发
  - 对比型
module: 07_AQS
created: 2026-04-18
---

# 公平锁 vs 非公平锁（公平锁检查队列，非公平锁直接CAS抢锁）

## 先说结论

ReentrantLock 支持公平锁和非公平锁两种模式。**非公平锁**（默认）允许新线程插队 CAS 抢锁，吞吐量更高；**公平锁**检查同步队列中是否有等待者，有则排队，保证 FIFO 但吞吐量较低。实际开发中默认用非公平锁。

## 深度解析

### 创建方式

```java
Lock unfairLock = new ReentrantLock();              // 默认非公平
Lock unfairLock = new ReentrantLock(false);         // 显式非公平
Lock fairLock = new ReentrantLock(true);             // 公平锁
```

### tryAcquire 差异

```
非公平 NonfairSync:
  lock()
    ├── state==0 → CAS(0,1) → 直接抢！不看队列
    │   ├── 成功 → 获取锁（插队成功）
    │   └── 失败 → acquire() → 入队排队
    └── state!=0 → 检查重入

公平 FairSync:
  lock()
    ├── state==0 → hasQueuedPredecessors()?
    │   ├── 有等待者 → acquire() → 入队排队
    │   └── 无等待者 → CAS(0,1)
    │       ├── 成功 → 获取锁
    │       └── 失败 → 入队排队
    └── state!=0 → 检查重入
```

### hasQueuedPredecessors

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    // h != t: 队列不为空
    // (s = h.next) == null: 正在入队
    // s.thread != current: 队列第一个等待者不是当前线程
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### 性能对比

```
场景：3 个线程竞争
                    非公平锁                    公平锁
T1: lock → 获取    lock → 获取               lock → 获取
T2: lock → 排队    lock → CAS抢失败→排队     lock → 排队
T3: lock → 排队    lock → CAS抢失败→排队     lock → 排队
T1: unlock         unlock                      unlock
    → 唤醒T2           → 唤醒T2                   → 唤醒T2
T2: 获取锁         T2: 获取锁                  T2: 获取锁
T3: 获取锁         T3: 获取锁                  T3: 获取锁

非公平锁额外开销：T2/T3 的 CAS 抢锁失败
公平锁额外开销：每次 tryAcquire 多一次 hasQueuedPredecessors 检查

总体：非公平锁吞吐量更高（减少线程切换）
      公平锁减少饥饿（但增加线程切换）
```

### 对比总结

| 维度 | 非公平锁 | 公平锁 |
|------|---------|--------|
| 插队 | ✅ 新线程可以 CAS 插队 | ❌ 必须排队 |
| 吞吐量 | **更高** | 较低 |
| 线程切换 | 较少 | 较多（保证排队） |
| 饥饿可能 | ✅ 可能（连续插队） | ❌ 不太可能 |
| 默认 | ✅ ReentrantLock 默认 | 需显式指定 |
| 适用场景 | 通用场景 | 严格公平要求 |

## 易错点/踩坑

- ❌ 公平锁保证绝对公平——`hasQueuedPredecessors` 有竞态条件，极端情况下仍可能插队
- ❌ 公平锁性能一定差——低竞争时差异很小
- ✅ synchronized 只有非公平模式

## 代码示例

```java
// 公平 vs 非公平锁的行为差异
ReentrantLock fairLock = new ReentrantLock(true);
ReentrantLock unfairLock = new ReentrantLock(false);

// 公平锁：T1释放后一定是T2获取（假设T2先排队）
// 非公平锁：T1释放后T3可能CAS抢到（即使T2已排队）
```

## 关联知识点

