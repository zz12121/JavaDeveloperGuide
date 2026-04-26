---
title: StampedLock
tags:
  - Java/并发
  - 原理型
module: 08_Lock
created: 2026-04-26
---

# StampedLock（JDK8 读写锁改进版）

## 先说结论

`StampedLock` 是 JDK8 引入的**读写锁改进版**，通过**乐观读**解决了 ReadWriteLock 的写锁饥饿问题。核心思想：读取时不阻塞写入，写入时阻塞所有读取。

**三种锁模式**：
- **写锁**（独占）：`writeLock()`，返回戳
- **悲观读锁**（共享）：`readLock()`，返回戳
- **乐观读**（无锁）：`tryOptimisticRead()`，返回一个戳，后续验证

## 深度解析

### 三种锁模式对比

| 模式 | 获取方式 | 并发能力 | 适用场景 |
|------|----------|---------|---------|
| **写锁** | `writeLock()` | 独占（阻塞所有读写） | 数据写入 |
| **悲观读锁** | `readLock()` | 共享（阻塞写锁） | 需要强一致性的读取 |
| **乐观读** | `tryOptimisticRead()` | 无锁（不阻塞任何线程） | 读多写少，数据一致性要求不高 |

### 核心思想：戳（stamp）

StampedLock 用一个 64 位 `long` 作为"戳"，每次获取锁时生成新的戳：

```
初始 stamp = 0
获取写锁：stamp = 写戳（奇数）
获取读锁：stamp = 读戳（偶数）
乐观读：stamp = 验证戳
```

**乐观读 vs 悲观读的本质区别**：

```
悲观读 lock.readLock()：
  线程 T1 ──→ [读取数据] ──→ [释放读锁]
  线程 T2 ──→ [等待 T1 释放读锁] ──→ [读取数据]
  问题：写入被读锁阻塞 → 写锁饥饿

乐观读 tryOptimisticRead()：
  线程 T1 ──→ [获取戳] ──→ [读取数据] ──→ [验证戳] ──→ 成功/重试
  线程 T2 ──→ [获取写锁] ──→ [写入数据] ──→ [释放写锁]
  优势：读取不阻塞写入 → 无饥饿
```

### 内部结构

```
StampedLock
├── state: long（64位戳）
│   ├── 低7位：写锁计数（WBIT = 0x80）
│   ├── 第8位：读锁计数（RBIT = 0xFF）
│   └── 其他位：版本号（每次写锁释放+1）
├── WBIT = 0x80（1000 0000）
├── RBIT = 0x7F（0111 1111）
└── 三个队列（CLH变体）
    ├── 写锁等待队列（writerQueue）
    ├── 读锁等待队列（readerQueue）
    └── 读转写队列（readersWithPad）
```

### API 详解

```java
StampedLock lock = new StampedLock();

// 写锁（独占）
long stamp = lock.writeLock();    // 阻塞直到获取
try {
    // 写入数据
} finally {
    lock.unlockWrite(stamp);      // 用对应的戳释放
}

// 悲观读锁（共享，阻塞写锁）
long stamp = lock.readLock();     // 阻塞直到获取
try {
    // 读取数据
} finally {
    lock.unlockRead(stamp);
}

// 乐观读（无锁，最快）
long stamp = lock.tryOptimisticRead();  // 立即返回戳
// 读取数据（此时可能有写锁在执行）
if (!lock.validate(stamp)) {            // 验证戳是否被修改
    // 戳被修改（期间有写锁）→ 降级为悲观读
    stamp = lock.readLock();
    try {
        // 重新读取
    } finally {
        lock.unlockRead(stamp);
    }
}
// 戳未被修改 → 数据一致，直接使用
```

### 锁转换方法

StampedLock 支持锁之间的相互转换：

```java
// 读锁 → 写锁（writeLock 与 readLock 互斥）
long stamp = lock.readLock();
try {
    // 读取数据
    if (需要写入) {
        long ws = lock.tryConvertToWriteLock(stamp);  // 尝试升级
        if (ws != 0) {
            stamp = ws;   // 升级成功，ws 成为新的有效戳
            // 写入数据
        } else {
            // 升级失败（有其他读锁）→ 解锁读锁，再获取写锁
            lock.unlockRead(stamp);
            stamp = lock.writeLock();
            // 写入数据
            lock.unlockWrite(stamp);
            return;
        }
    }
} finally {
    lock.unlockRead(stamp);
}
```

| 转换方法 | 成功条件 | 失败处理 |
|---------|---------|---------|
| `tryConvertToWriteLock(stamp)` | 无其他读锁持有 | 返回0，需手动重试 |
| `tryConvertToReadLock(stamp)` | 无写锁持有 | 返回0，需手动重试 |

### 典型代码模式

#### 乐观读模式（推荐）

```java
class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // 乐观读模式（大多数场景）
    double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // 获取戳
        double dx = x, dy = y;                   // 读取（可能不一致）
        if (!lock.validate(stamp)) {              // 验证
            stamp = lock.readLock();              // 降级为悲观读
            try {
                dx = x; dy = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(dx * dx + dy * dy);
    }

    // 写锁模式
    void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

#### 锁降级模式

```java
long stamp = lock.writeLock();
try {
    x = updateX();
    y = updateY();
    // 锁降级：写锁 → 读锁（数据可见性）
    long rs = lock.tryConvertToReadLock(stamp);
    if (rs != 0) {
        stamp = rs;
        // 继续持有读锁，安全读取 x 和 y
    }
} finally {
    lock.unlock(stamp);  // 解锁当前持有的锁（读锁或写锁）
}
```

### StampedLock vs ReadWriteLock

| 维度 | ReadWriteLock | StampedLock |
|------|--------------|-------------|
| 读锁类型 | 悲观读（共享） | 乐观读 + 悲观读 |
| 写锁饥饿 | 有（读锁阻塞写锁） | **无**（乐观读不阻塞写入） |
| 锁升级 | 不支持 | `tryConvertToWriteLock` 支持 |
| 锁降级 | 支持（标准降级模式） | 支持（通过 tryConvertToReadLock） |
| 可重入 | 可重入（ReentrantReadWriteLock） | **不可重入** |
| 公平模式 | 支持 | 支持（`StampedLock(boolean)`） |
| 性能 | 一般 | 乐观读场景性能最优 |
| 复杂度 | 简单 | 较高（需处理乐观读验证） |

> ⚠️ **重要限制**：StampedLock **不支持重入**！同一线程获取写锁后再次获取写锁会死锁。

## 易错点/踩坑

- ❌ **不可重入**：StampedLock 不支持同一线程重复获取同一锁
- ❌ **死锁**：在持有写锁时调用 `readLock()`/`tryOptimisticRead()` 会死锁
- ❌ **乐观读不保证一致性**：必须通过 `validate(stamp)` 验证
- ❌ **无限循环验证**：乐观读验证失败后应降级为悲观读，不应无限重试验证
- ✅ 写操作用 `writeLock()`，读操作用 `tryOptimisticRead()` + `validate()`
- ✅ 锁升级用 `tryConvertToWriteLock()`，失败时手动重试

## 代码示例

```java
// 场景：缓存实现（读多写少）
class Cache {
    private final StampedLock lock = new StampedLock();
    private Map<String, Object> cache = new HashMap<>();

    // 读取缓存（乐观读）
    Object get(String key) {
        long stamp = lock.tryOptimisticRead();
        Object value = cache.get(key);
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                value = cache.get(key);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return value;
    }

    // 更新缓存（写锁）
    void put(String key, Object value) {
        long stamp = lock.writeLock();
        try {
            cache.put(key, value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

## 关联知识点

- [AQS 15_ReentrantReadWriteLock](/11_并发/07_AQS/15_ReentrantReadWriteLock/ReentrantReadWriteLock.md) — ReadWriteLock 实现，state 分片，读写互斥规则
- [08_Lock 11_ReentrantReadWriteLock](/11_并发/08_Lock/11_ReentrantReadWriteLock/ReentrantReadWriteLock.md) — 读写锁的 API 使用和锁降级模式
- [AQS 12_可重入锁](/11_并发/07_AQS/12_可重入锁/可重入锁.md) — 可重入机制对比（StampedLock 不可重入）
