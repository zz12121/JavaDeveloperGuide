---
title: ReadWriteLock特性
tags:
  - Java/并发
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# ReadWriteLock特性

## 核心结论

ReadWriteLock 接口定义了读写锁的规范：`readLock()` 返回读锁（Lock），`writeLock()` 返回写锁（Lock）。核心特性是**读读共享、读写互斥、写写互斥、支持可重入、支持锁降级**。

## 深度解析

### 接口定义

```java
public interface ReadWriteLock {
    Lock readLock();   // 获取读锁
    Lock writeLock();  // 获取写锁
}
```

### 核心特性

#### 1. 读读共享

多个线程可同时持有读锁，不阻塞彼此。适用于读多写少场景。

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();

// 线程A
readLock.lock();
try { /* 读取 */ } finally { readLock.unlock(); }

// 线程B（不阻塞）
readLock.lock();
try { /* 读取 */ } finally { readLock.unlock(); }
```

#### 2. 读写互斥 / 写写互斥

读锁和写锁、写锁和写锁之间互斥：

```java
Lock writeLock = rwLock.writeLock();

// 持有读锁时，写锁获取阻塞
readLock.lock();
writeLock.lock(); // 阻塞

// 持有写锁时，读锁获取阻塞
writeLock.lock();
readLock.lock(); // 阻塞
```

#### 3. 可重入

- **读锁可重入**：同一线程可多次获取读锁
- **写锁可重入**：同一线程可多次获取写锁
- **写锁→读锁可重入**：持有写锁时可以获取读锁（锁降级前提）

#### 4. 锁降级

```
持有写锁 → 获取读锁 → 释放写锁 → 持有读锁
```

保证数据更新后能安全读取最新值，其他线程不能在中间插入写操作。

#### 5. 锁升级不支持

```
持有读锁 → 获取写锁 → ❌ 死锁
```

### 适用场景

| 场景 | 适合度 | 原因 |
|------|--------|------|
| 读多写少（缓存） | ⭐⭐⭐⭐⭐ | 读不阻塞，并发度高 |
| 读多写多 | ⭐⭐ | 写锁竞争激烈，优势不大 |
| 写多读少 | ⭐ | 不如直接用 ReentrantLock |
| 读多写极少 | ⭐⭐⭐⭐⭐ | 最佳场景，如本地缓存 |

### 性能对比

```
ReentrantLock:  所有操作串行化
ReadWriteLock:  读操作并行，写操作串行
  读并发度: O(n) 个读线程可同时进行
  写并发度: O(1) 同一时刻只有1个写线程
```

在 80%读 + 20%写的场景下，ReadWriteLock 吞吐量约为 ReentrantLock 的 5~10 倍。

## 关联知识点

