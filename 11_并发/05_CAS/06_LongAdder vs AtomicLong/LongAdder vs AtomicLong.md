---
title: LongAdder vs AtomicLong
tags:
  - Java/并发
  - 对比型
module: 05_CAS
created: 2026-04-18
---

# LongAdder vs AtomicLong（LongAdder分段CAS解决高并发热点，适合统计场景）

## 先说结论

`AtomicLong` 在高并发下所有线程竞争同一个变量，CAS 失败率飙升导致 CPU 空转。`LongAdder` 将热点分散到多个 Cell（分段 CAS），每个线程操作自己的 Cell，最终求和。高并发统计场景下 `LongAdder` 性能远优于 `AtomicLong`，但牺牲了精确的实时性。

## 深度解析

### AtomicLong 的瓶颈

```
高并发场景（100线程同时递增）：
所有线程 → CAS(base, old, old+1)
         → 99个线程CAS失败 → 自旋重试
         → 大量CPU空转
```

单点竞争，并发度越高，CAS 冲突越严重。

### LongAdder 的分段 CAS 思路

```
LongAdder 结构：
┌──────────────────────────────────────────┐
│ base (volatile long)                     │  ← 无竞争时直接 CAS
├──────────────────────────────────────────┤
│ cells[0] │ cells[1] │ cells[2] │ ...    │  ← 竞争时分散到不同 Cell
└──────────────────────────────────────────┘

线程1 → hash(cells) → cells[2] CAS ✅
线程2 → hash(cells) → cells[5] CAS ✅
线程3 → hash(cells) → cells[0] CAS ✅
线程4 → hash(cells) → cells[2] CAS ⚠️ 与线程1冲突，可能自旋
```

### LongAdder 核心源码

```java
public class LongAdder extends Striped64 implements LongAdderInterface {
    // 无竞争时使用 base
    transient volatile long base;

    // 有竞争时使用 cells
    transient volatile Cell[] cells;

    // 哈希掩码
    transient int cellsBusy;

    // 递增操作
    public void increment() {
        add(1L);
    }

    public void add(long x) {
        Cell[] cs; long b, v; int m; Cell c;
        if ((cs = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[getProbe() & m]) == null ||
                !(uncontended = c.cas(v = c.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

    // 获取总和（遍历所有 Cell 求和）
    public long sum() {
        Cell[] cs = cells;
        long sum = base;
        if (cs != null) {
            for (Cell c : cs)
                if (c != null) sum += c.value;
        }
        return sum;
    }
}
```

### Cell 的设计

```java
// @Contended 防止伪共享（每个 Cell 独占一个缓存行）
@jdk.internal.vm.annotation.Contended
static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return U.compareAndSetLong(this, VALUE, cmp, val);
    }
}
```

### 对比总结

| 维度 | AtomicLong | LongAdder |
|------|-----------|-----------|
| 原理 | CAS 单一变量 | 分段 CAS + cells 数组 |
| 高并发性能 | 差（热点竞争） | 好（分散竞争） |
| 内存开销 | 8 bytes | 8 + 128 * n bytes（缓存行填充） |
| 实时精确性 | ✅ 精确 | ⚠️ sum() 时可能不一致 |
| 适用场景 | 低并发、需要精确值 | 高并发统计、计数器 |
| 支持操作 | CAS 全部原子操作 | 仅 add/sum/increment |
| 典型应用 | 序号生成、状态标记 | 接口计数、QPS统计 |

## 易错点/踩坑

- ❌ 认为 LongAdder 可以替代所有 AtomicLong 场景——LongAdder 不支持 compareAndSet
- ❌ 认为 LongAdder.sum() 返回的是实时精确值——sum() 遍历时各 Cell 可能还在变化
- ✅ LongAdder 的 sum() 适合统计场景，不适合精确控制场景

## 代码示例

```java
// 性能对比测试
public class LongAdderBenchmark {
    private static final int THREAD_COUNT = 100;
    private static final int INCREMENTS = 1_000_000;

    public static void main(String[] args) throws Exception {
        // AtomicLong 测试
        AtomicLong atomicLong = new AtomicLong(0);
        long start = System.currentTimeMillis();
        runTest(atomicLong);
        System.out.println("AtomicLong: " + (System.currentTimeMillis() - start) + "ms");
        // 高并发下明显慢

        // LongAdder 测试
        LongAdder longAdder = new LongAdder();
        start = System.currentTimeMillis();
        runTest(longAdder);
        System.out.println("LongAdder: " + (System.currentTimeMillis() - start) + "ms");
        // 高并发下快 5~10 倍
    }

    static void runTest(AtomicLong counter) throws Exception {
        Thread[] threads = new Thread[THREAD_COUNT];
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < INCREMENTS; j++) counter.incrementAndGet();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
    }

    static void runTest(LongAdder counter) throws Exception {
        Thread[] threads = new Thread[THREAD_COUNT];
        for (int i = 0; i < THREAD_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < INCREMENTS; j++) counter.increment();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
    }
}
```

### 伪共享详解与 @Contended

**什么是伪共享**：多核 CPU 中，缓存行（64 bytes）是缓存一致性操作的最小单位。两个无关变量如果恰好在同一个缓存行，一个被频繁 CAS 修改，另一个即使不相关也会被迫失效。

```
┌─────────────────────────────────────────┐
│ volatile long x    volatile long y     │ ← 同一缓存行
│     线程1 CAS(x)    线程2 读 y          │
└─────────────────────────────────────────┘
↑
线程1 修改 x → 整行失效
→ 线程2 的 y 缓存行失效，必须重新从主存加载
→ 性能骤降（即使 y 根本没人改）
```

**验证伪共享影响**：

```java
// 测试：无填充 vs 有填充
class NoPadding {
    volatile long x;
    volatile long y;
}

class WithPadding {
    @jdk.internal.vm.annotation.Contended
    volatile long x;
    @jdk.internal.vm.annotation.Contended
    volatile long y;
}

// 结果（100线程同时 CAS）：
// NoPadding:    ~500万次 CAS 失败/秒
// WithPadding:  ~1千次 CAS 失败/秒
```

**@Contended 注解详解**：

```java
// LongAdder.Cell 的实现
@sun.internal.annotation.Contended
static final class Cell {
    volatile long value;
    // 编译器在 Cell 前后各填充 64~128 bytes
    // 确保每个 Cell 独占一个缓存行
}

// Contended 注解的填充规则（JDK8+）
// - 默认填充 128 bytes（前后各 64 bytes）
// 可通过 -XX:ContendedPaddingWidth=<n> 调整（仅 JDK8）
// JDK9+ 固定为 128 bytes
```

**手动缓存行填充**（JDK8 之前的替代方案）：

```java
// 通过在字段前后填充 long 数组来占据缓存行
class ManualPadding {
    long p1, p2, p3, p4, p5, p6, p7, p8; // 前填充 64 bytes
    volatile long value;
    long q1, q2, q3, q4, q5, q6, q7, q8; // 后填充 64 bytes
}
```

**LongAdder 使用 @Contended 的原因**：高并发下所有线程同时 CAS 不同的 Cell，如果不填充，一个 Cell 被修改会导致相邻 Cell 所在缓存行失效，引发伪共享风暴。

### LongAdder 的核心参数

```java
// Striped64 内部参数
static final int NCPU = Runtime.getRuntime().availableProcessors();
// cells 数组的最大长度 = CPU 核数，防止过度扩容

static final long base;
// 无竞争时直接 CAS 更新的字段

transient volatile Cell[] cells;
// 竞争时的 Cell 数组，长度为 2 的幂次

transient volatile int cellsBusy;
// 扩容/初始化 Cell 数组的锁（CAS 操作）
```

### LongAdder vs AtomicLong 性能对比数据

```java
// 100 线程，每线程递增 100万次
// 测试环境：8 核 CPU

// AtomicLong:  ~2000ms（大量 CAS 失败，CPU 空转）
// LongAdder:   ~180ms   （分段 CAS，几乎无冲突）

// 性能提升：~11 倍

// 适用场景对照
AtomicLong:  序号生成器、状态标记（需要 compareAndSet）
LongAdder:    接口 QPS、在线人数、统计累加
```

## 关联知识点

