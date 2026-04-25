---
title: ThreadLocalRandom与SplittableRandom
tags:
  - Java/并发
  - 随机数
  - 问答
module: 12_其他核心概念
created: 2026-04-25
---

# ThreadLocalRandom与SplittableRandom

## Q1：Random 类是线程安全的吗？为什么？

**A：**
Random 在**概念上**是线程安全的（使用 CAS），但在**性能上**不是线程安全的。

每个 Random 实例内部有一个 `AtomicLong seed`，多线程调用 `nextInt()` 时需要执行 `compareAndSet()` 来更新种子。在高并发下，大量线程同时 CAS 竞争同一个 `AtomicLong`，导致**性能急剧下降**。

JDK 源码中的 `nextInt()`：
```java
private final AtomicLong seed;
protected int next(int bits) {
    long oldseed, nextseed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));  // CAS 竞争！
    return (int)(nextseed >>> (48 - bits));
}
```

这就是为什么**多线程不要用 new Random()**。

---

## Q2：ThreadLocalRandom 是如何解决多线程竞争问题的？

**A：**
ThreadLocalRandom 为**每个线程维护独立的 Random 实例**（种子），彻底消除了线程竞争：

```java
// ThreadLocalRandom.current() 的简化实现
public static ThreadLocalRandom current() {
    Thread t = Thread.currentThread();
    ThreadLocalRandom rand = threadLocalRandomHolder.get(t);
    if (rand == null) {
        rand = initializeLocalRandom();
        threadLocalRandomHolder.set(t, rand);
    }
    return rand;
}

// 每个线程的 seed 是独立的，不需要 CAS
public int nextInt(int bound) {
    long m = threadLocalRandomSeed;  // 本地变量，无竞争
    long r = mix64(m);
    threadLocalRandomSeed = r;
    return (int)(r % bound);
}
```

线程之间各有独立种子，**调用时无需任何同步操作**，性能远高于 Random。

---

## Q3：ThreadLocalRandom.current() 和 new Random() 的区别？

**A：**

| 维度 | `new Random()` | `ThreadLocalRandom.current()` |
|------|----------------|------------------------------|
| 种子共享 | 所有线程共享同一个 `AtomicLong` | 每个线程独立种子 |
| 线程安全实现 | CAS 竞争 | 线程本地，无竞争 |
| 多线程性能 | 差（大量 CAS 失败重试） | 好 |
| 子线程 | 需要单独创建 | 自动获取各自种子 |
| 调用方式 | `new Random()` | 必须用 `ThreadLocalRandom.current()` |

---

## Q4：什么场景下用 SplittableRandom 而不是 ThreadLocalRandom？

**A：**
SplittableRandom 专为 **fork/join 框架和并行流** 设计，支持 `split()` 操作。

使用场景：
1. **并行流中每个任务需要独立随机数**：并行流的每个任务调用 `split()` 得到独立的生成器
2. **需要可预测的随机数序列**：fork/join 中，split 后的子生成器与父生成器有确定性的种子关系
3. **性能敏感**：split 操作比创建新的 SplittableRandom 更快

```java
// 场景：并行计算，每个任务独立随机数
SplittableRandom main = new SplittableRandom();

list.parallelStream().forEach(item -> {
    SplittableRandom sr = main.split();  // 每个任务得到独立生成器
    int random = sr.nextInt(100);  // 无竞争
});
```

---

## Q5：为什么并行流推荐使用 SplittableRandom？

**A：**
并行流使用 fork/join 框架分解任务。问题在于：

1. 如果用 `ThreadLocalRandom.current()`：每个线程的种子独立，但 fork 出来的子任务可能继承了父任务的种子（虚拟线程场景）
2. 如果用 `new Random()`：大量 CAS 竞争，性能极差

SplittableRandom 的 `split()` 解决了这个问题：
- 每个 split 生成**完全独立的新生成器**
- 种子之间没有关联，无法通过一个种子推断另一个
- 比 new SplittableRandom() 创建新实例更高效（内部有优化）

---

## Q6：ThreadLocalRandom 的子线程会继承父线程的种子吗？

**A：**
**不会**。ThreadLocalRandom 的种子是线程本地的（类似 ThreadLocal），每个线程调用 `current()` 时获取或初始化自己的种子，与父线程完全独立。这正是与 ThreadLocal 数据传递的关键区别——ThreadLocal 允许父线程设置值后被子线程继承，而 ThreadLocalRandom 每个线程都是完全独立的随机数生成器。

---

## Q7：SecureRandom 和 ThreadLocalRandom 的区别？

**A：**

| 维度 | ThreadLocalRandom | SecureRandom |
|------|-------------------|--------------|
| 用途 | 普通随机数 | 密码学安全 |
| 速度 | 非常快（纳秒级） | 慢（涉及系统熵源） |
| 随机性 | 伪随机（数学算法） | 密码学安全随机 |
| 线程竞争 | 无（线程独立） | 有（通常需要同步） |
| 适用场景 | 游戏、抽奖、业务随机 | 密码、令牌、会话ID |
