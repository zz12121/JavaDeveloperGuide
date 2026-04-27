
# StampedLock

## 核心结论

StampedLock 是 JDK8 引入的**读写锁增强版**，支持**乐观读**模式。乐观读不加锁，只获取一个"票据（stamp）"，读完后通过 validate 校验，期间如果有写操作则升级为悲观读锁重试。相比 ReentrantReadWriteLock，在**读多写少**场景下性能显著提升。

## 深度解析

### 三种模式

| 模式 | 方法 | 说明 |
|------|------|------|
| **写锁（悲观）** | `writeLock()` / `tryWriteLock()` | 独占锁，阻塞其他读写 |
| **读锁（悲观）** | `readLock()` / `tryReadLock()` | 与写互斥，与其他读共享 |
| **乐观读** | `tryOptimisticRead()` | 无锁读取，获取 stamp 校验 |

### StampedLock vs ReentrantReadWriteLock

| 对比 | StampedLock | ReentrantReadWriteLock |
|------|------------|----------------------|
| 乐观读 | ✅ 支持 | ❌ 不支持 |
| 可重入 | ❌ 不可重入 | ✅ 可重入 |
| Condition | ❌ 不支持 | ✅ 支持 |
| 锁降级 | ✅ 支持 | ✅ 支持 |
| 锁升级 | ❌ 不支持（写锁获取时没有持有读锁） | ❌ 不支持 |
| 性能 | 更高（乐观读无锁） | 较低（读锁也阻塞写） |
| stamp 机制 | 返回 long stamp | 无 |

### 核心工作流程

```
乐观读流程：
1. long stamp = tryOptimisticRead()  → 获取 stamp
2. 读取数据（无锁）
3. if (!validate(stamp))             → 校验期间是否有写
4.     升级为悲观读锁重读             → readLock() → 读取 → unlockRead()

写锁流程：
1. long stamp = writeLock()  → 获取写锁
2. 修改数据
3. unlockWrite(stamp)        → 释放写锁
```

### 内部实现

- 使用一个 **long state**（64位）：高 7 位表示锁状态，低 57 位表示版本号（序列）
- `writeLock()`：CAS 将 state 从 0 设为写锁标记
- `readLock()`：CAS 递增读计数（饱和递增防止溢出）
- `tryOptimisticRead()`：直接读取当前 state（获取版本号）
- `validate(stamp)`：比较当前 state 与 stamp 是否一致

## 代码示例

### 标准使用模板（重点）

```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 写操作
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // 乐观读（推荐）
    double distanceFromOrigin() {
        // 1. 获取乐观读 stamp
        long stamp = sl.tryOptimisticRead();
        // 2. 无锁读取
        double currentX = x, currentY = y;
        // 3. 校验：期间是否有写操作
        if (!sl.validate(stamp)) {
            // 4. 校验失败，升级为悲观读锁
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

### 锁转换

```java
// 读锁转写锁
long stamp = sl.readLock();
try {
    while (needWrite()) {
        long writeStamp = sl.tryConvertToWriteLock(stamp);
        if (writeStamp != 0L) {
            stamp = writeStamp; // 转换成功
            break;
        } else {
            sl.unlockRead(stamp);
            stamp = sl.writeLock(); // 转换失败，获取写锁
        }
    }
    // 执行写操作
} finally {
    sl.unlock(stamp); // 根据当前 stamp 类型自动释放
}
```

## 注意事项

- **不可重入**：同一个线程不能重复获取同一把锁，会死锁
- **不支持 Condition**：无法与 Condition 配合使用
- **不要在乐观读中调用可能阻塞的方法**：如 IO 操作、其他锁操作
- **JVM 建议**：大部分场景下 ReentrantReadWriteLock 已够用，StampedLock 适合读远多于写且性能敏感的场景

## 易错点与踩坑

### 1. 不可重入：最容易踩的坑

```java
StampedLock lock = new StampedLock();

// ❌ 同一个线程重复获取写锁，会死锁
long stamp = lock.writeLock();
try {
    doWrite();
    // 业务逻辑中再次尝试获取写锁
    long stamp2 = lock.writeLock();  // 死锁！
    try {
        doAnotherWrite();
    } finally {
        lock.unlockWrite(stamp2);
    }
} finally {
    lock.unlockWrite(stamp);  // 永远不会执行
}

// ✅ 正确做法：使用 ReentrantReadWriteLock（可重入）
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.writeLock().lock();
try {
    rwLock.writeLock().lock();  // 可重入，不会死锁
    try {
        doAnotherWrite();
    } finally {
        rwLock.writeLock().unlock();
    }
} finally {
    rwLock.writeLock().unlock();
}
```

### 2. 乐观读 vs 悲观读的选型错误

```java
StampedLock lock = new StampedLock();

// ❌ 误以为乐观读总是最好的
long stamp = lock.tryOptimisticRead();  // 获取乐观读 stamp
// ⚠️ 在这个stamp验证之前，数据可能被其他线程修改

if (!lock.validate(stamp)) {  // 验证失败
    // 升级为悲观读
    stamp = lock.readLock();
    try {
        readData();
    } finally {
        lock.unlockRead(stamp);
    }
}

// ✅ 正确选择：
// 1. 读多写少 + 冲突少 → 乐观读（性能最好）
// 2. 读多写少 + 冲突多 → 悲观读（避免频繁重试）
// 3. 读写均衡 → ReentrantReadWriteLock（可重入）
// 4. 需要 Condition → ReentrantReadWriteLock
```

### 3. writeLock 饥饿问题

```java
StampedLock lock = new StampedLock();

// ❌ 高并发读 + 少量写 → 写线程可能永久等待
while (true) {
    long stamp = lock.readLock();
    try {
        readData();  // 大量读线程抢占读锁
    } finally {
        lock.unlockRead(stamp);
    }
}

// 如果此时有写线程需要写锁：
long writeStamp = lock.writeLock();  // 饥饿！

// ✅ 解决：使用 stampedLock.promoteToWrite() 升级
// 或者使用 ReadWriteLock 的写锁优先模式
// 或者限制读线程的数量
```

### 4. validate 的误用

```java
StampedLock lock = new StampedLock();

// ❌ validate 只是检查这一刻stamp是否有效
long stamp = lock.tryOptimisticRead();

readStep1();  // 读了一些数据

// ⚠️ validate 在这里是 true，但后面的数据可能是脏的
if (lock.validate(stamp)) {
    readStep2();  // ❌ 可能读到不一致的数据！
}

// ✅ 正确做法：读完所有数据后再验证
long stamp = lock.tryOptimisticRead();
readAllData();  // 先读完所有需要的数据
if (!lock.validate(stamp)) {
    // 重新用悲观读
    stamp = lock.readLock();
    try {
        readAllData();
    } finally {
        lock.unlockRead(stamp);
    }
}
```

## 关联知识点
