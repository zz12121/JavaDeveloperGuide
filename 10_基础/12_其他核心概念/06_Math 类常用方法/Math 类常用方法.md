---
title: Math类常用方法
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

Math 类提供基本的数学运算方法，所有方法均为 **static**，Math 类本身是 final 的不可继承。JDK 8 新增了 `Math.floorMod`、`Math.fma` 等方法。

---

## 深度解析

### 1. 常用方法分类

#### 取整方法
```java
Math.ceil(3.2)    // 4.0   — 向上取整
Math.floor(3.8)   // 3.0   — 向下取整
Math.round(3.5)   // 4     — 四舍五入（返回 long）
Math.round(3.4)   // 3
Math.rint(3.5)    // 4.0   — 四舍五入（返回 double，银行家舍入）
Math.rint(2.5)    // 2.0   — 注意：2.5 的 rint 是 2.0（向偶数舍入）
```

#### 最值与绝对值
```java
Math.max(3, 5)          // 5
Math.min(3, 5)          // 3
Math.abs(-5)            // 5
Math.abs(Integer.MIN_VALUE) // 溢出！结果仍为负数
```

#### 幂与开方
```java
Math.pow(2, 3)     // 8.0     — 2的3次方
Math.sqrt(16)      // 4.0     — 平方根
Math.cbrt(27)      // 3.0     — 立方根
Math.hypot(3, 4)   // 5.0     — √(3² + 4²)
```

#### 随机数
```java
Math.random()  // [0.0, 1.0) 的 double
```

#### 其他
```java
Math.PI              // 3.141592653589793
Math.E               // 2.718281828459045
Math.log(Math.E)     // 1.0  — 自然对数
Math.log10(100)      // 2.0  — 以10为底的对数
Math.exactAdd(a, b)  // JDK 8+ 精确加法，溢出抛异常
Math.floorMod(a, b)  // JDK 8+ 总是返回非负余数
```

### 2. 注意事项

```java
// ❌ Math.abs(Integer.MIN_VALUE) 溢出
Math.abs(Integer.MIN_VALUE); // -2147483648（负数！）

// ✅ 使用 Math.abs 时要注意边界
// ✅ 需要0~n的随机整数：int num = (int)(Math.random() * n)
```

---

## 关联知识点
