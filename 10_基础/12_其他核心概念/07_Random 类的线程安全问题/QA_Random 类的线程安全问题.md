---
title: Random类的线程安全问题
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Random类的线程安全问题
## Q1：Random 是线程安全的吗？为什么多线程下性能差？

**A**：Random 是线程安全的，使用 CAS（compareAndSet）更新共享的 seed 原子变量。
但多线程下性能差的原因是**所有线程竞争同一个 AtomicLong 变量**：
- 线程越多，CAS 冲突越严重
- 冲突的线程需要不断自旋重试
- 高并发下大部分 CPU 时间浪费在 CAS 自旋上

---

## Q2：如何解决多线程下 Random 的性能问题？

**A**：使用 `ThreadLocalRandom`（JDK 7+），每个线程持有独立的 Random 实例和种子，完全消除了竞争。
```java
// ❌ 多线程共享 Random
private static final Random random = new Random();
int n = random.nextInt(100);

// ✅ ThreadLocalRandom（推荐）
int n = ThreadLocalRandom.current().nextInt(100);
```
> ThreadLocalRandom 不能通过 new 创建，只能通过 `ThreadLocalRandom.current()` 获取当前线程的实例。

# ThreadLocalRandom

## Q1：ThreadLocalRandom 为什么比 Random 快？

**A**：因为 **ThreadLocalRandom 的种子是线程私有的**，每个线程操作自己的种子变量，不需要 CAS 竞争共享变量。
Random 的所有线程竞争同一个 AtomicLong seed → 高并发下大量 CAS 自旋失败 → 性能下降。ThreadLocalRandom 完全消除了这个瓶颈。

---

## Q2：ThreadLocalRandom 如何使用？

**A**：
```java
// 获取实例（不是通过 new）
ThreadLocalRandom r = ThreadLocalRandom.current();

// 常用方法
r.nextInt(100);           // [0, 100)
r.nextInt(10, 20);        // [10, 20)
r.nextLong(1000);         // [0, 1000)
r.nextDouble();           // [0.0, 1.0)
r.nextBoolean();          // true/false
```
> 不能通过 `new ThreadLocalRandom()` 创建，只能通过 `ThreadLocalRandom.current()` 获取。