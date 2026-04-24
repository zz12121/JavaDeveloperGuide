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
```
