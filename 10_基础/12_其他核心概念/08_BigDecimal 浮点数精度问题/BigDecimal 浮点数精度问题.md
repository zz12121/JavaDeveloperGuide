---
title: BigDecimal浮点数精度问题
tags:
  - 场景型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

浮点数（float/double）在计算机中采用 IEEE 754 标准，某些十进制小数无法精确表示，导致精度丢失。涉及**金额计算**时必须使用 `BigDecimal`。

---

## 深度解析

### 1. 浮点数精度问题

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004
System.out.println(1.0 - 0.9);          // 0.09999999999999998
System.out.println(2.00 - 1.10);        // 0.8999999999999999
```
原因：IEEE 754 浮点数用二进制表示，0.1 在二进制中是无限循环小数，无法精确存储。

### 2. BigDecimal 正确用法

#### ❌ 错误方式
```java
// 不要用 double 构造 BigDecimal
BigDecimal a = new BigDecimal(0.1);      // 0.1000000000000000055511151231257827021181583404541015625
BigDecimal b = new BigDecimal("0.1");    // 0.1（精确）
```

#### ✅ 正确方式
```java
// 1. 用字符串构造（最推荐）
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
a.add(b);  // 0.3

// 2. 用 BigDecimal.valueOf()（内部转为字符串）
BigDecimal c = BigDecimal.valueOf(0.1);  // 0.1
```

### 3. BigDecimal 运算

```java
BigDecimal a = new BigDecimal("10.5");
BigDecimal b = new BigDecimal("3");

a.add(b);                  // 13.5
a.subtract(b);             // 7.5
a.multiply(b);             // 31.5

// ⚠️ divide 必须指定精度和舍入模式，否则可能抛 ArithmeticException
a.divide(b, 2, RoundingMode.HALF_UP);  // 3.50

// 比较：不要用 equals（精度不同会返回 false）
// 使用 compareTo
a.compareTo(b);  // 正数表示 a > b，0 相等，负数 a < b
```

### 4. 舍入模式

| 模式 | 说明 | 示例 1.5 | 示例 2.5 |
|------|------|---------|---------|
| `HALF_UP` | 四舍五入 | 2 | 3 |
| `HALF_DOWN` | 五舍六入 | 1 | 2 |
| `HALF_EVEN` | 银行家舍入（向偶数） | 2 | 2 |
| `UP` | 远离零方向 | 2 | 3 |
| `DOWN` | 向零方向 | 1 | 2 |
| `CEILING` | 向正无穷 | 2 | 3 |
| `FLOOR` | 向负无穷 | 1 | 2 |

### 5. 金融场景最佳实践

```java
// 金额计算统一使用 HALF_UP，保留2位小数
public static BigDecimal add(BigDecimal a, BigDecimal b) {
    return a.add(b).setScale(2, RoundingMode.HALF_UP);
}

// 使用 int/long 表示分/厘（避免 BigDecimal 开销）
// 10.50 元 → 1050 分
long amount = 1050L; // 分
```

---

## 关联知识点

