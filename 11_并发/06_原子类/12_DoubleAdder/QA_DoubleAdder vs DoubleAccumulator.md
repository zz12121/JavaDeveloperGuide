---
title: DoubleAdder vs DoubleAccumulator
tags:
  - Java/并发
  - 问答
  - 场景型
module: 06_原子类
created: 2026-04-26
---

# DoubleAdder vs DoubleAccumulator（高并发 double 累加）

## Q1：DoubleAdder 和 LongAdder 的实现有什么区别？

**A**：两者原理完全相同，都基于 `Striped64` 的分段 CAS 机制。区别在于处理的数据类型：

| 维度 | LongAdder | DoubleAdder |
|------|-----------|-------------|
| 数据类型 | long | double |
| Cell 存储 | long value | long value（double 的位表示） |
| CAS 操作 | `Unsafe.compareAndSetLong` | `Unsafe.compareAndSetLong`（位比较） |
| 精度 | 精确 | IEEE 754 双精度精确 |

```java
// DoubleAdder 内部：double → long 位表示 → CAS
// 累加时：newValue = Double.longBitsToDouble(cell.value) + x
// CAS 时：cell.cas(oldBits, Double.doubleToRawLongBits(newValue))
```

---

## Q2：为什么不用 AtomicReference<Double> 来做 double 累加？

**A**：`AtomicReference<Double>` 虽然线程安全，但每次累加都要创建一个新的 `Double` 对象，性能极差：

```java
// ❌ AtomicReference<Double>：每次 add 创建新对象
AtomicReference<Double> ref = new AtomicReference<>(0.0);
for (int i = 0; i < 1000; i++) {
    Double old = ref.get();
    ref.compareAndSet(old, old + 1.5); // 每次创建新 Double 对象
}

// ✅ DoubleAdder：直接操作 double 位，无对象创建
DoubleAdder adder = new DoubleAdder();
for (int i = 0; i < 1000; i++) {
    adder.add(1.5); // 无对象创建，高性能
}
```

**性能差距**：在 100 线程并发场景下，`AtomicReference<Double>` 的吞吐量约为 `DoubleAdder` 的 **1/50**。

---

## Q3：volatile double 可以做并发累加吗？为什么？

**A**：**不可以**。`volatile double` 保证可见性，但不保证原子性：

```java
// ❌ volatile double 不是线程安全的
volatile double total = 0.0;

public void add(double amount) {
    // 这行代码不是原子的！
    // 等价于：1) 读取 total → 2) 计算 total + amount → 3) 写回 total
    total += amount;  // 多线程下会丢失更新！
}

// 正确做法
DoubleAdder total = new DoubleAdder();
public void add(double amount) {
    total.add(amount);  // 线程安全
}
```

> **面试加分**：很多候选人认为 `volatile` 可以解决并发问题，但实际上 `volatile` 只解决了可见性，复合操作（如 `+=`）需要 CAS 或锁来保证原子性。

---

## Q4：DoubleAdder 和 DoubleAccumulator 的区别是什么？

**A**：与 `LongAdder` vs `LongAccumulator` 的关系完全一致：

```java
// DoubleAdder = DoubleAccumulator(x, y) -> x + y
DoubleAdder adder = new DoubleAdder();
DoubleAccumulator accum = new DoubleAccumulator(Double::sum, 0.0); // 等价

// DoubleAccumulator 支持自定义运算
DoubleAccumulator maxAccum =
    new DoubleAccumulator(Double::max, Double.NEGATIVE_INFINITY);
maxAccum.accumulate(3.14); // 记录最大值
maxAccum.accumulate(2.71);
System.out.println(maxAccum.get()); // 3.14

// DoubleAccumulator 的运算同样必须满足结合律
// Math.max 满足结合律：max(max(a,b),c) == max(a,max(b,c))
```

---

## Q5：DoubleAdder 的 sum() 是原子快照吗？会有什么问题？

**A**：不是原子快照。`sum()` 在读取 base 和遍历 Cell[] 的过程中，其他线程可能还在累加：

```java
// ⚠️ 非原子快照场景
DoubleAdder adder = new DoubleAdder();

// 线程1：读取 sum
double snapshot = adder.sum();  // 读到 base + 部分 Cell

// 线程2：同时累加
adder.add(1.5);  // 可能修改了一个还未读取的 Cell

// 结果：snapshot 不是完整的快照
```

**实际影响**：在绝大多数业务场景下，误差可以忽略。但如果需要精确快照，需要加锁：

```java
// 精确快照方案
private final DoubleAdder adder = new DoubleAdder();
private final ReentrantLock lock = new ReentrantLock();
private double snapshot;

public void record(double value) {
    adder.add(value);
}

public double getSnapshot() {
    lock.lock();
    try {
        snapshot = adder.sum();
        return snapshot;
    } finally {
        lock.unlock();
    }
}
```

---

## Q6：DoubleAdder 在实际项目中有哪些典型应用？

**A**：

| 场景 | 示例 |
|------|------|
| 金融系统 | 订单金额累加、折扣统计 |
| 性能监控 | 响应时间求和、DB 时间统计 |
| 数据采集 | 传感器数据聚合（温度、压力） |
| 限流器 | 令牌桶的剩余令牌数 |

```java
// 典型：计算平均响应时间
public class HttpMonitor {
    private final DoubleAdder totalTime = new DoubleAdder();
    private final LongAdder count = new LongAdder();

    public void record(long durationMs) {
        totalTime.add(durationMs);
        count.increment();
    }

    public double avgDuration() {
        return totalTime.sum() / count.sum();
    }
}
```

## 关联知识点
