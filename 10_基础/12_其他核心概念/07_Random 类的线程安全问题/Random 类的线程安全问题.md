---
title: Random类的线程安全问题
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
difficulty: 易
---

## 核心结论

`java.util.Random` 是**线程安全**的（使用 CAS 更新种子），但在**高并发场景下性能很差**——多线程竞争同一个 AtomicLong 种子会导致大量 CAS 自旋重试。高并发推荐使用 `ThreadLocalRandom`。

---

## 深度解析

### 1. Random 的线程安全机制

```java
// Random.nextInt() 的简化逻辑
public int nextInt() {
    return next(32); // CAS 更新 seed
}

protected int next(int bits) {
    long oldSeed, nextSeed;
    AtomicLong seed = this.seed;
    do {
        oldSeed = seed.get();
        nextSeed = (oldSeed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldSeed, nextSeed)); // CAS 竞争
    return (int)(nextSeed >>> (48 - bits));
}
```

每次生成随机数都通过 **CAS** 更新共享的 seed 原子变量：
- 单线程：没问题
- 多线程：多个线程同时 CAS 同一个变量，大量自旋重试

### 2. 性能问题

```
Thread1 → CAS(seed) → 成功
Thread2 → CAS(seed) → 失败 → 重试 → 成功
Thread3 → CAS(seed) → 失败 → 重试 → 失败 → 重试 → 成功
...     → CAS(seed) → 大量重试
```

线程越多，CAS 冲突越严重，吞吐量急剧下降。

### 3. 解决方案

| 方案 | 说明 |
|------|------|
| `ThreadLocalRandom` | 每个线程持有独立种子，无竞争（推荐） |
| 每个线程 new Random() | 可行但不优雅 |
| 加锁 | 性能更差，不推荐 |

```java
// ❌ 多线程共享 Random
private static final Random random = new Random();

// ✅ 使用 ThreadLocalRandom
int num = ThreadLocalRandom.current().nextInt(100);
```

---

# ThreadLocalRandom
## 核心结论

`ThreadLocalRandom`（JDK 7+）是 Random 的多线程增强版，每个线程维护独立的种子，消除了 CAS 竞争。在高并发随机数生成场景下性能远优于 Random。

---

## 深度解析

### 1. 工作原理

```
Thread1 → seed1 → 无竞争
Thread2 → seed2 → 无竞争
Thread3 → seed3 → 无竞争
```

- 种子存储在 Thread 的 ThreadLocalRandomProbe 字段中
- 每个线程操作自己的种子，无需 CAS 同步
- 初始化时使用一个全局 seed 生成各线程的初始种子

### 2. 使用方式

```java
// 获取当前线程的 ThreadLocalRandom
ThreadLocalRandom r = ThreadLocalRandom.current();

// 常用方法
r.nextInt();              // 随机 int
r.nextInt(bound);         // [0, bound)
r.nextInt(origin, bound); // [origin, bound)
r.nextLong(bound);        // [0, bound)
r.nextDouble();           // [0.0, 1.0)
r.nextBoolean();          // true/false

// 连续生成
int a = r.nextInt(100);
int b = r.nextInt(100);
```

### 3. 性能对比

|特性|Random|ThreadLocalRandom|
|---|---|---|
|线程安全|CAS 竞争|无竞争（线程隔离）|
|单线程性能|较好|接近|
|多线程性能|差|**好（接近单线程 Random）**|
|使用方式|`new Random()`|`ThreadLocalRandom.current()`|

### 4. 注意事项

```java
// ❌ 不能直接 new
// new ThreadLocalRandom(); // 编译错误

// ✅ 通过 current() 获取
ThreadLocalRandom.current().nextInt(100);

// ⚠️ 在 ForkJoinPool 等线程池中，线程可能被复用，
// 但 ThreadLocalRandom 的种子会随线程一起正确工作
```

## 关联知识点
