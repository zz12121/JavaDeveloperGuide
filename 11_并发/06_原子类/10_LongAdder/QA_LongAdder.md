---
title: LongAdder
tags:
  - Java/并发
  - 问答
  - 源码型
module: 06_原子类
created: 2026-04-18
---

# LongAdder（分段CAS + Cells数组，高并发统计场景首选）

## Q1：LongAdder 是怎么解决 AtomicLong 热点竞争的？

**A**：通过**分段 CAS**（Cells 数组）将竞争分散：

1. **无竞争**：直接 CAS 更新 `base` 字段
2. **有竞争**：创建 Cells 数组，每个线程通过 `probe & (length-1)` 哈希到不同 Cell
3. **Cell 内 CAS**：线程只 CAS 自己的 Cell，减少冲突
4. **扩容**：冲突过多时 cells 数组翻倍（最大约 CPU 核数 × 2）
5. **求和**：`sum()` 遍历 base + 所有 Cell 值

每个 Cell 用 `@Contended` 独占缓存行，避免伪共享。

---

## Q2：LongAdder 为什么高并发性能好？

**A**：关键在于**分散热点**：

- `AtomicLong`：100 个线程竞争 1 个变量 → 99 个 CAS 失败 → 大量自旋
- `LongAdder`：100 个线程分散到 ~8 个 Cell（8核CPU）→ 每 Cell ~12.5 个线程 → 冲突大幅减少

在高并发下，LongAdder 性能通常是 AtomicLong 的 5~10 倍。

---

## Q3：LongAdder 的 sum() 为什么不是精确的？

**A**：`sum()` 遍历 base + 所有 Cell 时，其他线程可能正在更新：

```java
public long sum() {
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null) sum += c.value;  // 遍历期间值可能变化
    }
    return sum;
}
```

- 适合**统计场景**（QPS、计数器），误差可以接受
- 不适合**精确控制场景**（如限流精确到 1）

---

## Q4：LongAdder 支持哪些操作？不支持哪些？

**A**：

**支持**：`add(x)`、`increment()`、`decrement()`、`sum()`、`reset()`、`sumThenReset()`

**不支持**：`compareAndSet()`、`get()`（无精确单值）、`getAndAdd()`、`getAndIncrement()`

本质是**累加器**，不是通用的原子变量。

---

## Q5：LongAdder 的 Cell 为什么用 @Contended？它是怎么避免伪共享的？

**A**：每个 Cell 的 `value` 字段用 `@Contended` 填充，使得每个 Cell 独占一个缓存行（64 bytes）：

```java
// 没有 @Contended 的后果
class BadCell {
    volatile long value;  // 和相邻 Cell 的 value 挤在同一缓存行
}
// 线程1 CAS Cell[0].value → Cell[1].value 所在缓存行失效
// 线程2 读 Cell[1].value → 缓存 miss，被迫重新加载
// → 伪共享：Cell[0] 的修改拖累了 Cell[1] 的性能

// @Contended 后的效果
@sun.internal.annotation.Contended
class GoodCell {
    volatile long value;  // 前后各填充 128 bytes，独占缓存行
}
// 线程1 CAS Cell[0].value → 只有 Cell[0] 的行失效
// 线程2 CAS Cell[1].value → 完全不受影响，因为在不同缓存行
```

**验证伪共享影响**（禁用 @Contended 后测试）：
```bash
# 禁用 @Contended
-XX:-RestrictContended

# 性能结果：禁用后高并发 CAS 失败率提升 ~100 倍
```

---

## Q6：LongAdder 的 Cells 数组是如何初始化的？扩容机制是什么？

**A**：

```java
// 初始化过程（Striped64.longAccumulate）
1. cells == null → 尝试 CAS base（无竞争时直接用 base）
2. base CAS 失败 → 创建 cells = new Cell[2]
3. 每个 Cell 初始化为 Cell(0)

扩容触发条件：
- Cell 的 CAS 连续失败 2 次（探测到竞争）
- cells.length < NCPU * 2（最大约 CPU 核数 × 2）

// 扩容过程
Cell[] newCells = new Cell[newLength];
for (int i = 0; i < newLength; i++) {
    newCells[i] = cells[i]; // 迁移旧 Cell
}
cells = newCells; // 替换

// 扩容成本：数组迁移 + 新建对象
// 所以 LongAdder 适合"写多读少"场景，写入达到稳态后不再扩容
```

**为什么最大长度是 CPU 核数 × 2？**
- 超过 CPU 核数的 Cell 没有意义（同时运行的线程数 ≤ 核数）
- 限制长度减少内存开销和伪共享风险

## 关联知识点