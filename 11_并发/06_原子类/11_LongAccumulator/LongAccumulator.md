---
title: LongAccumulator
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# LongAccumulator（LongAdder通用版，支持自定义运算）

## 先说结论

`LongAccumulator`（JDK8）是 `LongAdder` 的通用版本，支持自定义累加函数（不限于加法）。内部同样基于 `Striped64` 的分段 CAS 机制，但允许传入 `LongBinaryOperator` 定义任意二元运算。`LongAdder` 本质上就是 `LongAccumulator(x, y) -> x + y` 的特例。

## 深度解析

### 核心 API

```java
// 构造：初始值 + 累加函数
LongAccumulator accum = new LongAccumulator(Long::max, Long.MIN_VALUE);

// 累加操作（将当前值与新值进行运算）
accum.accumulate(value);

// 获取当前累加结果
long result = accum.get();

// 重置
accum.reset();
```

### 与 LongAdder 的关系

```java
// LongAdder 等价于
LongAdder adder = new LongAdder();
// 等价于
LongAccumulator accum = new LongAccumulator((x, y) -> x + y, 0);

// LongAdder 源码
public class LongAdder extends Striped64 {
    public void add(long x) { longAccumulate(x, null, uncontended); }
    public long sum() { /* 同 LongAccumulator.get() */ }
    // 实际上 LongAdder 的 longAccumulate 传入 null 作为 accumulatorFunction
    // 内部默认使用 x + y
}
```

### 典型应用场景

```java
// 场景1：求最大值
LongAccumulator maxAccum = new LongAccumulator(Long::max, Long.MIN_VALUE);
maxAccum.accumulate(10);
maxAccum.accumulate(25);
maxAccum.accumulate(18);
System.out.println(maxAccum.get()); // 25

// 场景2：求最小值
LongAccumulator minAccum = new LongAccumulator(Math::min, Long.MAX_VALUE);
minAccum.accumulate(10);
minAccum.accumulate(5);
minAccum.accumulate(18);
System.out.println(minAccum.get()); // 5

// 场景3：乘法
LongAccumulator mulAccum = new LongAccumulator((x, y) -> x * y, 1);
mulAccum.accumulate(2);
mulAccum.accumulate(3);
mulAccum.accumulate(4);
System.out.println(mulAccum.get()); // 24

// 场景4：求幂次（记录最高幂次）
LongAccumulator powAccum = new LongAccumulator(
    (current, newVal) -> newVal > current ? newVal : current, 0
);
```

### 内部实现

```java
public class LongAccumulator extends Striped64 {
    private final LongBinaryOperator function;  // 自定义运算
    private final long identity;                  // 初始值

    // 累加操作
    public void accumulate(long x) {
        Cell[] cs; long b, v, r; int m; Cell c;
        if ((cs = cells) != null
            || (r = function.applyAsLong(b = base, x)) != b
            && !casBase(b, r)) {
            // 有竞争，分散到 Cell
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0
                || (c = cs[getProbe() & m]) == null
                || !(uncontended = (r = function.applyAsLong(
                    v = c.value, x)) == v || c.cas(v, r)))
                longAccumulate(x, function, uncontended);
        }
    }

    // 获取结果（汇总 base + 所有 Cell）
    public long get() {
        Cell[] cs = cells;
        long result = base;
        if (cs != null) {
            for (Cell c : cs)
                if (c != null)
                    result = function.applyAsLong(result, c.value);
        }
        return result;
    }
}
```

### LongAdder vs LongAccumulator

| 维度 | LongAdder | LongAccumulator |
|------|-----------|-----------------|
| 运算类型 | 固定为加法 | 自定义 LongBinaryOperator |
| 初始值 | 默认 0 | 自定义 identity |
| 性能 | 略好（无函数调用开销） | 略差（每次需调用 function） |
| 适用场景 | 计数、统计 | max/min/自定义聚合 |

## 易错点/踩坑

- ❌ 认为累加函数可以随意写——函数必须满足结合律（`(a op b) op c == a op (b op c)`），否则 Cell 之间的合并顺序影响结果
- ❌ 初始值不正确——`Long::max` 的初始值应是 `Long.MIN_VALUE`，不是 0
- ✅ `accumulate(x)` 是将 x 与当前值进行运算，不是替换当前值

## 代码示例

```java
// 并发统计：最大响应时间 + 总请求数
public class ResponseStats {
    private final LongAccumulator maxResponseTime =
        new LongAccumulator(Long::max, 0);
    private final LongAdder totalRequests = new LongAdder();
    private final LongAdder totalResponseTime = new LongAdder();

    public void recordResponse(long responseTimeMs) {
        maxResponseTime.accumulate(responseTimeMs);
        totalRequests.increment();
        totalResponseTime.add(responseTimeMs);
    }

    public long getMaxResponseTime() { return maxResponseTime.get(); }
    public double getAvgResponseTime() {
        long total = totalRequests.sum();
        return total == 0 ? 0 : (double) totalResponseTime.sum() / total;
    }
}
```

## 关联知识点