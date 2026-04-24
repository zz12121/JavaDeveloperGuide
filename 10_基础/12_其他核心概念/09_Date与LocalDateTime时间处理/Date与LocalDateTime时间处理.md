---
title: Date与LocalDateTime时间处理
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

JDK 8 引入了 `java.time` 包（JSR 310），替代了老旧的 `Date`/`Calendar`。新 API **不可变、线程安全、API 设计清晰**，是处理日期时间的推荐方式。

---

## 深度解析

### 1. 旧 API 的问题

```java
// Date：可变、非线程安全、年份从1900开始、月份从0开始
Date date = new Date(2026, 3, 18); // 实际是 3926年5月！
Date date2 = new Date(); date2.setTime(0); // 可变，线程不安全

// Calendar：API 臃肿、也是可变的
Calendar cal = Calendar.getInstance(); // 时区依赖默认
```

### 2. 新 API（java.time）

#### 核心类

| 类 | 用途 | 示例 |
|---|------|------|
| `LocalDate` | 日期（无时间） | 2026-04-18 |
| `LocalTime` | 时间（无日期） | 13:30:00 |
| `LocalDateTime` | 日期+时间（无时区） | 2026-04-18T13:30:00 |
| `ZonedDateTime` | 带时区的日期时间 | 2026-04-18T13:30+08:00 |
| `Instant` | 时间戳（UTC） | 精确到纳秒 |
| `Duration` | 时间段（时分秒） | PT2H30M |
| `Period` | 时间段（年月日） | P1Y2M3D |

#### 常用操作

```java
// 创建
LocalDate today = LocalDate.now();
LocalDate date = LocalDate.of(2026, 4, 18);
LocalDateTime dateTime = LocalDateTime.now();

// 解析
LocalDate parsed = LocalDate.parse("2026-04-18");
LocalDateTime parsed2 = LocalDateTime.parse("2026-04-18T13:30:00");

// 修改（不可变，返回新对象）
LocalDate tomorrow = today.plusDays(1);
LocalDate lastMonth = today.minusMonths(1);

// 获取
today.getYear();        // 2026
today.getMonthValue();  // 4
today.getDayOfMonth();  // 18
today.getDayOfWeek();   // SATURDAY

// 格式化
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日");
today.format(formatter); // 2026年04月18日

// 时间戳转换
Instant instant = Instant.now();
long epochMilli = instant.toEpochMilli(); // 毫秒时间戳
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
```

### 3. 旧新 API 互转

```java
// Date ↔ Instant ↔ LocalDateTime
Date date = new Date();
Instant instant = date.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());

LocalDateTime now = LocalDateTime.now();
Instant inst = now.atZone(ZoneId.systemDefault()).toInstant();
Date legacyDate = Date.from(inst);
```

### 4. 对比

| 维度 | Date/Calendar | java.time |
|------|---------------|-----------|
| 可变性 | 可变（线程不安全） | **不可变（线程安全）** |
| 月份起始 | 从 0 开始 | **从 1 开始** |
| API 设计 | 臃肿，set(int field, int value) | **链式调用，清晰** |
| 时区处理 | 隐式，容易出错 | **显式** |

---

## 关联知识点

