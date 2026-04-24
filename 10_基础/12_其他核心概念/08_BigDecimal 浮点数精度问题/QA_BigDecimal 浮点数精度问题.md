---
title: BigDecimal浮点数精度问题
tags:
  - 场景型
  - Java/其他核心概念
  - 问答
module: 12_其他核心概念
created: 2026-04-18
---

# BigDecimal浮点数精度问题
## Q1：为什么 0.1 + 0.2 不等于 0.3？

**A**：计算机使用 IEEE 754 标准的二进制浮点数表示小数。0.1 在二进制中是一个无限循环小数，无法被精确存储，只能存储一个近似值。因此 0.1 + 0.2 的结果也是近似值 0.30000000000000004。
> 涉及金额等精度要求高的场景，必须使用 BigDecimal。

---

## Q2：BigDecimal 创建时为什么推荐用字符串而不是 double？

**A**：`new BigDecimal(0.1)` 使用的是 double 的近似值，本身就带有精度误差。`new BigDecimal("0.1")` 直接使用字符串的精确十进制表示。
```java
new BigDecimal(0.1)       // 0.10000000000000000555...（不精确）
new BigDecimal("0.1")     // 0.1（精确）
BigDecimal.valueOf(0.1)   // 0.1（精确，内部转为字符串）
```

---

## Q3：BigDecimal 的 equals 和 compareTo 有什么区别？

**A**：
- **equals**：同时比较值和精度（scale）。`new BigDecimal("1.0").equals(new BigDecimal("1.00"))` 返回 **false**
- **compareTo**：只比较数值，忽略精度。`new BigDecimal("1.0").compareTo(new BigDecimal("1.00"))` 返回 **0**
```java
// 比较大小始终用 compareTo
if (a.compareTo(b) == 0) { /* 相等 */ }
if (a.compareTo(b) > 0)  { /* a > b */ }
```
