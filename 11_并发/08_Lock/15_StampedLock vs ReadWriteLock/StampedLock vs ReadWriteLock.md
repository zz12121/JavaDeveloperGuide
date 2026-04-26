---
title: StampedLock vs ReadWriteLock
tags:
  - Java/并发
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# StampedLock vs ReadWriteLock

## 核心结论

StampedLock 通过乐观读消除了读锁对写锁的阻塞，在**读多写少**场景下性能远超 ReentrantReadWriteLock。但 StampedLock 不可重入、不支持 Condition，使用更复杂。

## 深度解析

### 性能对比

```
场景：100 个读线程 + 1 个写线程

ReentrantReadWriteLock：
  写线程等待所有读线程释放锁 → 写延迟高
  读线程之间共享锁 → 读吞吐高

StampedLock（乐观读）：
  读线程不加锁 → 写线程不被阻塞 → 写延迟低
  validate 失败时才升级为悲观读 → 大部分时候无锁
```

### 全面对比

| 维度 | ReentrantReadWriteLock | StampedLock |
|------|----------------------|------------|
| 乐观读 | ❌ | ✅ 核心特性 |
| 可重入 | ✅ | ❌ |
| Condition | ✅ newCondition() | ❌ |
| 公平锁 | ✅ 可选 | ❌ 非公平 |
| 锁降级 | ✅ 写→读 | ✅ writeLock→readLock |
| 锁升级 | ❌ | 部分支持 tryConvertToWriteLock |
| 内存排序 | happen-before 保证 | happen-before 保证 |
| 适用场景 | 通用 | 读远多于写 |
| 复杂度 | 低 | 较高 |

### 乐观读为什么更快

```
传统读写锁：
  读操作 → 获取读锁 → 其他线程写操作被阻塞 → 读完释放

乐观读：
  读操作 → 获取版本号 → 直接读数据 → 校验版本号
    → 版本没变 → 成功（全程无锁）
    → 版本变了 → 升级为悲观读锁重读
```

在大部分读操作都能一次 validate 通过的场景下，乐观读几乎没有锁竞争。

### 选择建议

| 场景 | 推荐 |
|------|------|
| 读远多于写，性能敏感 | StampedLock |
| 需要可重入或 Condition | ReentrantReadWriteLock |
| 通用场景，团队不熟悉 StampedLock | ReentrantReadWriteLock |
| 写多读少 | 两者差异不大，选 ReentrantReadWriteLock |

## 关联知识点