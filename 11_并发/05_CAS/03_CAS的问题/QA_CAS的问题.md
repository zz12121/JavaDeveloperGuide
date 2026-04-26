---
title: CAS的问题
tags:
  - Java/并发
  - 问答
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS的问题（ABA问题/循环时间长/只能保证一个变量原子性）

## Q1：CAS 有哪些缺点？

**A**：CAS 有三个经典问题：

1. **ABA 问题**：值从 A→B→A，CAS 误判为没有变化。解决方案：加版本号（`AtomicStampedReference`）
2. **自旋开销大**：竞争激烈时反复 CAS 失败，CPU 空转。解决方案：限制自旋次数、使用 `LongAdder` 分段
3. **只能保证单个变量原子性**：无法同时 CAS 更新多个变量。解决方案：封装为对象用 `AtomicReference`，或加锁

---

## Q2：什么是 ABA 问题？如何解决？

**A**：ABA 问题是指线程1读到的旧值是 A，在准备 CAS 期间，其他线程将值改为 B 又改回 A，线程1 CAS 时发现值仍是 A，认为没被修改过。

**危害场景**：链表/栈操作中，节点被删除后重新分配到同一地址，CAS 无法区分新旧节点。

**解决方案**：

```java
// AtomicStampedReference：带版本号的 CAS
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

int stamp = ref.getStamp();    // 获取版本号
ref.compareAndSet(10, 20, stamp, stamp + 1); // 同时比较值和版本号

// AtomicMarkableReference：带标记位
AtomicMarkableReference<Integer> ref = new AtomicMarkableReference<>(10, false);
ref.compareAndSet(10, 20, false, true); // 同时比较值和标记
```

---

## Q3：CAS 自旋开销如何优化？

**A**：

| 策略 | 说明 | 应用 |
|------|------|------|
| 自适应自旋 | JVM 根据上次自旋成功率动态调整次数 | 锁升级中的轻量级锁 |
| 分段 CAS | 将热点分散到多个 Cell | `LongAdder` |
| 阻塞退让 | 自旋一定次数后让出 CPU | `LockSupport.parkNanos()` |
| 批量操作 | 减少单次 CAS 的竞争 | `LongAdder.sum()` 时合并结果 |

# ABA问题

## Q1：什么是 ABA 问题？举个具体例子

**A**：假设主内存中值 V=10，线程1 读到 A=10 准备 CAS(10,20)。在执行前，线程2 先将值改为 11，又改回 10。线程1 CAS 时发现 V 仍是 10，认为没被修改，成功将 V 改为 20。

实际值被改过两次，但 CAS 检测不到。在链表、栈等数据结构中，节点被删除后重新分配到相同值的对象，可能导致结构损坏。

---

## Q2：AtomicStampedReference 和 AtomicMarkableReference 有什么区别？

**A**：

|维度|AtomicStampedReference|AtomicMarkableReference|
|---|---|---|
|额外信息|int 版本号（stamp）|boolean 标记（mark）|
|检测粒度|精确到第几次修改|只能知道"改没改过"|
|适用场景|需要完整修改历史|只关心"脏标记"|
|版本管理|每次修改 stamp++|标记在 true/false 间切换|

```java
// StampedReference：精确版本控制
int oldStamp = stampedRef.getStamp();
stampedRef.compareAndSet(10, 20, oldStamp, oldStamp + 1);

// MarkableReference：二元标记
boolean oldMark = markableRef.isMarked();
markableRef.compareAndSet(10, 20, oldMark, !oldMark);
```

---

## Q3：实际开发中 ABA 问题真的会遇到吗？

**A**：取决于场景：

- **会遇到**：无锁数据结构（ConcurrentLinkedQueue、无锁栈）、CAS 操作对象引用而非基本类型
- **不会遇到**：`AtomicInteger` 简单计数器——值变回原值对业务无影响
- **JDK 内部**：`ConcurrentHashMap` 的 `ForwardingNode` 等使用了 `AtomicReferenceFieldUpdater`，不依赖版本号因为扩容场景不会 ABA

# 自旋CAS

## Q1：什么是自旋 CAS？什么时候适合用？

**A**：自旋 CAS 是 CAS 失败后在循环中不断重试，直到成功为止。适用于：

- **竞争不激烈**（1~3 个线程竞争）
- **临界区短**（简单读写操作，纳秒~微秒级）
- **不需要公平性**（不保证先到先得）

不适合：高竞争场景、临界区包含 IO 操作、需要严格公平性。

---

## Q2：自旋 CAS 浪费 CPU 怎么办？

**A**：优化策略：

1. **限制自旋次数**：超过阈值后 yield 或 park
2. **自适应自旋**（JDK6+）：JVM 自动根据历史成功率调整
3. **分段减少竞争**：`LongAdder` 将热点分散到多个 Cell
4. **退化为阻塞锁**：锁升级机制（轻量级锁→重量级锁）

```java
// 加入 yield 的优化自旋
int retries = 0;
do {
    old = value;
    new = old + 1;
    if (++retries > Runtime.getRuntime().availableProcessors()) {
        Thread.yield();
        retries = 0;
    }
} while (!CAS(old, new));
```

---

## Q3：JVM 的自适应自旋是怎么工作的？

**A**：JDK6 引入，JVM 根据上次锁获取的情况动态决定自旋时间：

- 如果上次自旋**成功**了 → 认为这次也大概率成功 → 适当增加自旋时间
- 如果上次自旋**失败了** → 认为竞争激烈 → 减少或跳过自旋，直接膨胀为重量级锁

这比 JDK5 固定自旋次数（`-XX:PreBlockSpin=10`）更智能。