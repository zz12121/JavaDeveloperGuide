---
title: LongAccumulator
tags:
  - Java/并发
  - 问答
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# LongAccumulator（LongAdder通用版，支持自定义运算）

## Q1：LongAccumulator 和 LongAdder 有什么区别？

**A**：

- `LongAdder`：只能做**加法**，初始值固定为 0
- `LongAccumulator`：支持**自定义运算**（`LongBinaryOperator`），可自定义初始值

```java
// LongAdder = LongAccumulator(x, y) -> x + y, identity=0
LongAdder adder = new LongAdder();
LongAccumulator accum = new LongAccumulator(Long::sum, 0); // 等价

// LongAccumulator 可以做其他运算
LongAccumulator max = new LongAccumulator(Long::max, Long.MIN_VALUE);
LongAccumulator mul = new LongAccumulator((x, y) -> x * y, 1);
```

---

## Q2：LongAccumulator 的运算函数有什么要求？

**A**：运算函数必须是**结合律的**（associative），即 `(a op b) op c == a op (b op c)`。

因为 `get()` 时需要按某种顺序合并 base 和各 Cell 的值，如果运算不满足结合律，合并顺序不同会导致不同结果。

- ✅ 满足结合律：加法、乘法、max、min、与或非
- ❌ 不满足结合律：减法、除法

---

## Q3：什么时候该用 LongAccumulator 而不是 LongAdder？

**A**：

| 场景 | 推荐 | 原因 |
|------|------|------|
| 接口计数 | LongAdder | 只需要加法，性能略好 |
| 最大/最小响应时间 | LongAccumulator | 需要自定义运算 |
| 累计乘积 | LongAccumulator | 需要乘法 |
| 自定义聚合 | LongAccumulator | 灵活性 |

简单说：**只加法用 LongAdder，其他运算用 LongAccumulator**。
