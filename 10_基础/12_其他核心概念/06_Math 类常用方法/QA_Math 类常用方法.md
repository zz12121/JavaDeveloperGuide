---
title: Math类常用方法
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Math类常用方法
## Q1：Math.ceil、floor、round 的区别？

**A**：
- **ceil**：向上取整，返回 double（`Math.ceil(3.2)` → `4.0`）
- **floor**：向下取整，返回 double（`Math.floor(3.8)` → `3.0`）
- **round**：四舍五入，返回 long（`Math.round(3.5)` → `4`）
注意返回类型不同：ceil/floor 返回 double，round 返回 long。

---

## Q2：Math.abs(Integer.MIN_VALUE) 返回什么？为什么？

**A**：返回 `Integer.MIN_VALUE`（-2147483648），仍然是**负数**。
原因：int 范围是 -2^31 ~ 2^31-1，正数最大值比负数最小值小 1，所以 `Integer.MIN_VALUE` 取绝对值后溢出，结果仍是自身。
```java
// 安全做法：转为 long 再取绝对值
Math.abs((long)Integer.MIN_VALUE); // 2147483648

---

## Q3：Math.random 和 Random 类生成的随机数范围是什么？

**A**：

| 方法 | 返回范围 | 示例 |
|------|----------|------|
| `Math.random()` | `[0.0, 1.0)` 左闭右开 | `0.723...` |
| `Random.nextInt()` | `[-2^31, 2^31-1)` | 任意 int |
| `Random.nextInt(n)` | `[0, n)` 左闭右开 | `[0, n)` |

```java
// 生成 [0, 100) 的随机整数
int random = (int)(Math.random() * 100);
int random2 = new Random().nextInt(100);

// 生成 [min, max] 的随机整数
int randomInRange = min + (int)(Math.random() * (max - min + 1));
```

---

## Q4：StrictMath 和 Math 的区别是什么？

**A**：
`StrictMath` 和 `Math` 的API几乎相同，但有本质区别：

| 维度 | Math | StrictMath |
|------|------|------------|
| **计算结果** | **允许不同 JVM 实现自行优化**，结果可能因平台而异 | **严格遵循 IEEE 754**，所有平台结果一致 |
| **性能** | 通常更快（可使用平台优化指令如 FMA） | 跨平台一致性优先，可能更慢 |
| **典型场景** | 日常业务计算 | 金融、科学计算（要求结果确定性） |

```java
// Math：JVM 可能使用 CPU 的 FMA 指令优化，结果可能略有不同
double a = Math.sin(1.0);

// StrictMath：保证跨平台结果完全一致
double b = StrictMath.sin(1.0);

// Math 使用平台优化（以空间换精度），StrictMath 使用纯 Java 实现
// 两者在大多数情况下结果相同，但在边界情况下可能有差异
```

> **选择建议**：除非需要严格的跨平台一致性（如科学计算、金融系统），否则优先使用 `Math`，性能更好。
```
