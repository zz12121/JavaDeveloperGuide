---
title: ReentrantLock特性
tags:
  - Java/并发
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock特性

## 核心结论

ReentrantLock 是基于 AQS 实现的可重入独占锁，相比 synchronized 提供了更丰富的锁特性：**可中断、可超时、公平/非公平切换、多个条件变量**。

## 深度解析

### 特性全景

| 特性 | 说明 |
|------|------|
| 可重入 | 同一线程可多次获取锁，state 递增；释放时 state 递减至 0 才完全释放 |
| 可中断 | `lockInterruptibly()` 获取锁时响应中断 |
| 可超时 | `tryLock(timeout)` 指定时间内获取锁，超时自动放弃 |
| 公平/非公平 | 构造时选择，公平锁按 FIFO 顺序，非公平锁允许插队 |
| 多条件变量 | `newCondition()` 创建多个等待队列，精确控制线程等待/唤醒 |
| tryLock 尝试 | `tryLock()` 非阻塞尝试获取，立即返回成功/失败 |

### 内部结构

```
ReentrantLock
├── sync: Sync（AQS子类）
│   ├── NonfairSync（默认，非公平）
│   └── FairSync（公平）
└── 条件变量列表
```

- `sync` 是 AQS 的子类，封装了 `tryAcquire` / `tryRelease` 逻辑
- 公平与非公平的区别在于 `tryAcquire` 的实现：公平锁检查 CLH 队列中是否有前驱节点

### 可重入机制

可重入通过 AQS 的 `state` 字段实现，详细原理见 **AQS 12_可重入锁**。

```
state 计数原理：
  首次获取：state 0 → 1，记录 exclusiveOwnerThread
  同线程重入：state +1（无需 CAS）
  释放：state -1，降至 0 时才唤醒后继
```

> 📖 **关联阅读**：[AQS 12_可重入锁](/11_并发/07_AQS/12_可重入锁/可重入锁.md) — monitor 方法、CLH 队列、重入实现完整源码分析。

## 代码示例

```java
ReentrantLock lock = new ReentrantLock(true); // 公平锁

lock.lock();
try {
    // 临界区
    // 同一线程可重入
    lock.lock();
    try {
        // 嵌套临界区
    } finally {
        lock.unlock();
    }
} finally {
    lock.unlock();
}

System.out.println("Hold count: " + lock.getHoldCount()); // 0（释放后）
System.out.println("Queue length: " + lock.getQueueLength());
System.out.println("Is fair: " + lock.isFair());
```

## 关联知识点
