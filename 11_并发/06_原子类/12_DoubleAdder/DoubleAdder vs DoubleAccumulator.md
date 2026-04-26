---
title: DoubleAdder vs DoubleAccumulator
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-26
---

# DoubleAdder vs DoubleAccumulator（高并发 double 累加）

## 先说结论

`DoubleAdder` 和 `DoubleAccumulator` 是 JDK8 新增的高并发浮点累加器，与 `LongAdder` 原理完全一致（基于 `Striped64` 分段 CAS），但专门处理 `double` 类型。`DoubleAdder` 只支持加法，`DoubleAccumulator` 支持自定义 `DoubleBinaryOperator`。

> **注意**：`DoubleAdder` 内部没有分段锁——因为 double 的位表示不是线性连续的（NaN、Infinity 等），无法用简单的 CAS 碰撞来分段。JDK 使用了与 `LongAdder` 相同的 Striped64 架构，但专门适配了 double 的位操作。

## 深度解析

### 核心 API

```java
// DoubleAdder：只支持加法
DoubleAdder adder = new DoubleAdder();
adder.add(1.5);           // 累加
double sum = adder.sum();  // 获取累加和
adder.reset();             // 重置为 0.0

// DoubleAccumulator：自定义运算
DoubleAccumulator accum =
    new DoubleAccumulator((x, y) -> Math.max(x, y), Double.NEGATIVE_INFINITY);
accum.accumulate(3.14);
accum.accumulate(2.71);
System.out.println(accum.get()); // 3.14
```

### 与 LongAdder 的对比

| 维度 | LongAdder | DoubleAdder |
|------|-----------|------------|
| 数据类型 | long | double |
| 分段机制 | Striped64 Cell[] | Striped64 Cell[]（long 存储 double 位表示） |
| 精度 | 精确 | 精确（IEEE 754 双精度） |
| add() 返回 | void | void |
| sum() 返回 | long | double |
| 性能 | 高并发最优 | 高并发最优 |

### DoubleAdder 内部实现原理

```java
// DoubleAdder 内部使用 Striped64，Cell 中存的是 double 的 long 位表示
// 因为 Cell.value 是 long，double 需要转换为位表示才能 CAS
static final class Cell {
    volatile long value;  // double 的 IEEE 754 位表示
    Cell(double x) { value = Double.doubleToRawLongBits(x); }
    // 累加时：
    // 1. 读取 Cell.value（位表示）
    // 2. double newVal = Double.longBitsToDouble(value) + x
    // 3. CAS 新位表示
}

// add(x) 实际执行
public void add(double x) {
    Cell[] cs; double b, v, r; int m; Cell c;
    if ((cs = cells) != null
        || (r = b + x) != b
        && !casBase(b = base, Double.doubleToRawLongBits(r))) {
        // 分段 CAS：分散到不同 Cell
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0
            || (c = cs[getProbe() & m]) == null
            || !(uncontended = (r = Double.longBitsToDouble(c.value) + x)
                     != (v = Double.longBitsToDouble(c.value))
                 || !c.cas(v, Double.doubleToRawLongBits(r))))
            doubleAccumulate(x, null, uncontended);
    }
}
```

### DoubleAccumulator 典型应用

```java
// 场景1：高并发金融统计（金额累加）
public class OrderStats {
    private final DoubleAdder totalAmount = new DoubleAdder();
    private final DoubleAdder totalDiscount = new DoubleAdder();

    public void recordOrder(double amount, double discount) {
        totalAmount.add(amount);
        totalDiscount.add(discount);
    }

    public double getNetAmount() {
        return totalAmount.sum() - totalDiscount.sum();
    }
}

// 场景2：最大延迟记录
DoubleAccumulator maxLatency =
    new DoubleAccumulator(Double::max, Double.NEGATIVE_INFINITY);

// 场景3：平均值计算（需配合 LongAdder 计数）
public class LatencyTracker {
    private final DoubleAdder totalLatency = new DoubleAdder();
    private final LongAdder count = new LongAdder();

    public void record(double latencyMs) {
        totalLatency.add(latencyMs);
        count.increment();
    }

    public double getAvgLatency() {
        return totalLatency.sum() / count.sum();
    }
}
```

### DoubleAdder vs volatile double vs AtomicReference<Double>

| 维度 | DoubleAdder | volatile double | AtomicReference<Double> |
|------|------------|-----------------|----------------------|
| 线程安全 | ✅ 原子累加 | ❌ 非原子（racy） | ✅ 原子替换 |
| 高并发性能 | ✅ 最优 | ⚠️ 有数据覆盖风险 | ❌ 每次创建新对象 |
| 精度 | ✅ IEEE 754 精确 | ✅ IEEE 754 精确 | ❌ Boxing/Unboxing 精度损失 |
| sum() | O(1) | O(1) | O(1) |

> **⚠️ volatile double 不是线程安全的**：在 `volatile double d` 上执行 `d += x` 不是一个原子操作，多线程并发时会丢失更新。

## 易错点/踩坑

- ❌ `volatile double` 做累加不是线程安全的——`d += x` 是 read-modify-write 复合操作
- ❌ `DoubleAdder` 不能直接做减法——需要配合另一个 `DoubleAdder` 累加负值
- ❌ `DoubleAccumulator` 的自定义函数也必须满足结合律，否则 `get()` 结果不可靠
- ✅ `DoubleAdder.sum()` 是非原子快照，读取期间其他线程的累加不会反映在本次结果中
- ✅ 浮点精度问题：`1.1 + 2.2 ≠ 3.3`（IEEE 754 二进制浮点固有误差）

## 代码示例

```java
// 典型高并发场景：多线程统计响应时间和
public class PerformanceMonitor {
    private final DoubleAdder totalResponseTime = new DoubleAdder();
    private final DoubleAdder totalDbTime = new DoubleAdder();
    private final LongAdder requestCount = new LongAdder();

    public void record(long responseTimeMs, long dbTimeMs) {
        totalResponseTime.add(responseTimeMs);
        totalDbTime.add(dbTimeMs);
        requestCount.increment();
    }

    public double getAvgResponseTime() {
        return totalResponseTime.sum() / requestCount.sum();
    }

    public double getAvgDbTime() {
        return totalDbTime.sum() / requestCount.sum();
    }

    public void snapshot() {
        double avgResp = getAvgResponseTime();
        double avgDb = getAvgDbTime();
        System.out.printf("Avg: %.2fms (DB: %.2fms)%n", avgResp, avgDb);
    }
}
```

## 关联知识点
