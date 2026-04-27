---
tags: [Java并发, 计数器, LongAdder, AtomicLong, Striped64, 伪共享]
module: 17_实际应用与场景
chapter: 03_并发计数器
---

# 并发计数器

## Q1：高并发计数用什么？

**A**：

高并发统计场景推荐 `LongAdder`：

```java
LongAdder counter = new LongAdder();
counter.increment();          // +1，无返回值
counter.add(10);              // +10
long total = counter.sum();   // 汇总（近似值）
counter.reset();              // 重置为 0
long sumAndReset = counter.sumThenReset(); // 原子汇总并重置
```

`LongAdder` 使用**分段 CAS + Cells 数组**，将竞争分散到多个 Cell 上，高并发下性能是 `AtomicLong` 的 5~10 倍。

---

## Q2：LongAdder 和 AtomicLong 怎么选？

**A**：

- **AtomicLong**：适合并发量中等（< 100 线程）、需要**精确实时计数**的场景
- **LongAdder**：适合高并发（≥ 100 线程）、**统计类场景**（如 QPS、在线人数）

| 维度 | AtomicLong | LongAdder |
|------|-----------|-----------|
| 精确度 | ✅ 精确（get() 是精确值） | ⚠️ sum() 是近似值（并发写入时） |
| 高并发性能 | 中（CAS 自旋） | 高（分段竞争） |
| 内存占用 | 低（1个long） | 较高（base + Cell[]） |
| 支持 CAS 更新 | ✅ compareAndSet | ❌ 不支持 |
| 适用场景 | 精确计数、CAS 操作 | 统计、QPS、访问量 |

注意：`LongAdder.sum()` 返回的值不是精确的（调用时刻可能有线程在更新），但对于统计场景足够准确。

---

## Q3：为什么 LongAdder 在高并发下比 AtomicLong 快？

**A**：

`AtomicLong` 所有线程竞争同一个 `value` 变量，CAS 失败则自旋重试，高并发下自旋次数激增。

`LongAdder` 将竞争分散（**Striped64 原理**）：
1. 无竞争时直接 CAS 修改 `base`
2. 有竞争时创建/扩展 `Cell[]` 数组，线程通过 `threadLocalRandomProbe` 哈希映射到不同 Cell 各自累加
3. 调用 `sum()` 时汇总 `base + 所有 Cell 值`
4. Cell 数组最大扩展到 CPU 核数，充分利用多核并行

**核心思想**：用**空间换时间**，将单点竞争转化为多点并行累加。

---

## Q4：LongAdder 的 Cell 是如何消除伪共享的？

**A**：

**伪共享问题**：CPU 缓存行通常 64 字节。若两个 `Cell` 对象恰好在同一缓存行，线程 A 修改 Cell[0]、线程 B 修改 Cell[1]，会导致彼此缓存失效，反复 MESI 协议同步，性能反而下降。

**解决方案**：`@Contended` 注解

```java
@sun.misc.Contended  // JVM 在对象前后填充字节，独占缓存行
static final class Cell {
    volatile long value;
    // JVM 自动在 value 前后填充约 64-128 字节
    // 确保每个 Cell 独占至少一条 64 字节缓存行
}
```

效果：不同线程操作不同 Cell 时，互不影响对方的 CPU 缓存，彻底消除伪共享。

**类比**：就像给每个人一个单独房间，而不是多人挤在一张桌子上——互不干扰。

---

## Q5：LongAccumulator 和 LongAdder 有什么区别？

**A**：

`LongAccumulator` 是 `LongAdder` 的泛化版本，支持**自定义累加函数**：

```java
// LongAdder：只能做加法，初始值为 0
LongAdder adder = new LongAdder();
adder.add(5);   // 累加

// LongAccumulator：可自定义运算，支持任意初始值
LongAccumulator maxAcc = new LongAccumulator(Long::max, Long.MIN_VALUE);
maxAcc.accumulate(100);  // 保留最大值
maxAcc.accumulate(200);
maxAcc.get();  // 200

LongAccumulator sumAcc = new LongAccumulator((x, y) -> x + y, 0);
sumAcc.accumulate(10);  // 等价于 LongAdder，但更灵活
```

**关键限制**：`LongAccumulator` 的函数必须满足**交换律和结合律**（加法、最大/最小值等），不能用于需要中间状态的操作。

`reset()` 会将 `accumulator` 重置为初始值，而非 0，这点和 `LongAdder` 不同。

---

## Q6：如何用 LongAdder 实现滑动窗口 QPS 统计？

**A**：

```java
// 场景：统计最近 1 分钟的请求量（滑动窗口）
public class QpsCounter {
    // 60个桶，每个桶代表1秒的请求数
    private final LongAdder[] buckets = new LongAdder[60];
    private final AtomicLong[] timestamps = new AtomicLong[60];

    public QpsCounter() {
        for (int i = 0; i < 60; i++) {
            buckets[i] = new LongAdder();
            timestamps[i] = new AtomicLong(0);
        }
    }

    public void increment() {
        long now = System.currentTimeMillis() / 1000;  // 当前秒
        int idx = (int)(now % 60);
        // 如果这个桶是旧数据，清零重用
        if (timestamps[idx].get() != now) {
            timestamps[idx].set(now);
            buckets[idx].reset();
        }
        buckets[idx].increment();
    }

    public long getLastMinuteQps() {
        long now = System.currentTimeMillis() / 1000;
        long total = 0;
        for (int i = 0; i < 60; i++) {
            // 只统计最近60秒内的桶
            if (now - timestamps[i].get() < 60) {
                total += buckets[i].sum();
            }
        }
        return total;
    }
}
```

**LongAdder 在此场景的优势**：高并发下并发写入单个桶时，远比 `synchronized` 或 `AtomicLong` 性能好。
