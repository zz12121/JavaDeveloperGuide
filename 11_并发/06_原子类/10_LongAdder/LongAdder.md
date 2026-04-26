---
title: LongAdder
tags:
  - Java/并发
  - 源码型
module: 06_原子类
created: 2026-04-18
---

# LongAdder（分段CAS + Cells数组，高并发统计场景首选）

## 先说结论

`LongAdder`（JDK8）通过**分段 CAS + Cells 数组**解决 `AtomicLong` 在高并发下的热点竞争问题。无竞争时直接 CAS 更新 `base`，有竞争时创建 cells 数组，每个线程哈希到不同的 Cell 独立累加，最终通过 `sum()` 汇总。高并发统计场景下性能远超 `AtomicLong`。

## 深度解析

### 核心结构

```
LongAdder 继承 Striped64
┌──────────────────────────────────────────────────┐
│ Striped64                                         │
│ ├── base: volatile long      (无竞争时直接CAS)     │
│ ├── cells: volatile Cell[]   (有竞争时分散累加)     │
│ ├── cellsBusy: volatile int  (cells数组初始化锁)   │
│ └── casBase / casCells / getProbe                  │
│                                                    │
│ Cell (@Contended 遲免伪共享)                         │
│ └── value: volatile long   (每个Cell独立CAS)       │
└──────────────────────────────────────────────────┘
```

### Cell 设计——避免伪共享

```java
@jdk.internal.vm.annotation.Contended  // 缓存行填充，避免伪共享
static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return U.compareAndSetLong(this, VALUE, cmp, val);
    }
}
```

`@Contended` 让每个 Cell 独占一个缓存行（64 bytes），避免多线程修改相邻 Cell 时的缓存失效。

### add() 核心流程

```java
public void add(long x) {
    Cell[] cs; long b, v; int m; Cell c;
    // 快速路径：无竞争时直接 CAS base
    if ((cs = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        // cells 未创建 或 哈希到的 Cell 为空 或 Cell CAS 失败
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[getProbe() & m]) == null ||
            !(uncontended = c.cas(v = c.value, v + x)))
            // 慢速路径：初始化 cells / 创建 Cell / 扩容
            longAccumulate(x, null, uncontended);
    }
}
```

### 执行流程图

```
add(x)
  │
  ├── cells == null?
  │   ├── YES → CAS(base, base+x)
  │   │   ├── 成功 → 返回
  │   │   └── 失败 → longAccumulate() → 创建 cells[2]
  │   │
  │   └── NO → probe & (length-1) 定位 Cell
  │       ├── Cell != null → CAS(cell.value, cell.value+x)
  │       │   ├── 成功 → 返回
  │       │   └── 失败 → longAccumulate() → 可能扩容
  │       │
  │       └── Cell == null → longAccumulate() → 创建新 Cell
  │
  └── longAccumulate() 内部逻辑：
      ├── 获取 cellsBusy 锁（CAS 0→1）
      ├── 创建/扩容 cells（2的幂次）
      ├── 扩容条件：CAS 冲突次数 ≥ CPU 核数
      └── 释放 cellsBusy
```

### sum() 方法

```java
public long sum() {
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null) sum += c.value;
    }
    return sum;
}
```

**注意**：`sum()` 不是原子操作，遍历时各 Cell 可能还在变化。适合统计场景，不适合精确控制。

### Cells 扩容

```
初始: cells = null → base CAS
第一次冲突: cells = Cell[2]
继续冲突且 CAS 失败次数 >= CPU 核数: cells = Cell[4]
继续冲突: cells = Cell[8]
...
最大: cells = Cell[CPU核数 × 2]（不超过预分配限制）
```

## 易错点/踩坑

- ❌ `sum()` 是精确值——sum() 遍历过程中值可能被修改
- ❌ LongAdder 支持 `compareAndSet`——不支持，它只做累加
- ❌ LongAdder 可以替代所有 AtomicLong——需要精确 CAS 的场景不行
- ✅ `sum()` 没有加锁，是最终一致性

## 代码示例

```java
// 接口 QPS 统计
public class QPSCounter {
    private final LongAdder requestCount = new LongAdder();
    private final LongAdder errorCount = new LongAdder();

    public void recordRequest() { requestCount.increment(); }
    public void recordError() { errorCount.increment(); }

    public long getTotalRequests() { return requestCount.sum(); }
    public long getTotalErrors() { return errorCount.sum(); }
    public double getErrorRate() {
        long total = requestCount.sum();
        return total == 0 ? 0 : (double) errorCount.sum() / total;
    }

    // 每秒重置
    public void reset() {
        requestCount.reset();
        errorCount.reset();
    }
}
```

## 关联知识点