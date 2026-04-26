---
title: Date与LocalDateTime时间处理
tags:
  - 问答
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Date与LocalDateTime时间处理
## Q1：JDK 8 的时间 API 相比 Date/Calendar 有什么优势？

**A**：
1. **不可变**：所有类都是不可变的，线程安全
2. **月份从 1 开始**：Calendar 月份从 0 开始，极易出错
3. **API 清晰**：链式调用 `today.plusDays(1).minusMonths(2)`
4. **时区处理明确**：ZonedDateTime 显式携带时区信息
5. **线程安全**：不需要额外同步

---

## Q2：LocalDate、LocalDateTime、Instant 的区别？

**A**：
- **LocalDate**：只有日期（年月日），无时间和时区
- **LocalTime**：只有时间（时分秒），无日期和时区
- **LocalDateTime**：日期 + 时间，无时区
- **ZonedDateTime**：日期 + 时间 + 时区
- **Instant**：UTC 时间戳（精确到纳秒），用于机器间通信
```java
Instant.now().toEpochMilli(); // 毫秒时间戳，等同 System.currentTimeMillis()
```

---

## Q3：DateTimeFormatter 有哪些常用模式字母？

**A**：

| 模式字母 | 含义 | 示例 |
|----------|------|------|
| `y` | 年 | 2026 |
| `M` | 月 | 04（4月） |
| `d` | 日 | 18 |
| `H` | 小时（24制） | 14 |
| `h` | 小时（12制） | 02 |
| `m` | 分钟 | 30 |
| `s` | 秒 | 45 |
| `S` | 毫秒 | 123 |
| `a` | AM/PM | 下午 |
| `E` | 星期 | 星期六 |

```java
// 常用格式化
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH:mm:ss");
String str = today.format(formatter); // "2026年04月18日 14:30:00"

// ISO 标准格式
DateTimeFormatter iso = DateTimeFormatter.ISO_LOCAL_DATE_TIME;
```

---

## Q4：如何处理不同时区之间的时间转换？

**A**：

```java
// 北京时间转纽约时间
LocalDateTime beijingTime = LocalDateTime.of(2026, 4, 18, 20, 0);
ZonedDateTime beijing = beijingTime.atZone(ZoneId.of("Asia/Shanghai"));
ZonedDateTime newYork = beijing.withZoneSameInstant(ZoneId.of("America/New_York"));
// newYork: 2026-04-18T08:00-04:00

// Instant 直接转换时区
Instant now = Instant.now();
ZonedDateTime utc = now.atZone(ZoneId.of("UTC"));
ZonedDateTime tokyo = now.atZone(ZoneId.of("Asia/Tokyo"));
```

---

## Q5：Duration 和 Period 的区别？

**A**：

| 维度 | Duration | Period |
|------|----------|--------|
| **适用对象** | 时间（秒、纳秒） | 日期（年、月、日） |
| **表示** | `PT2H30M` (2小时30分) | `P1Y2M3D` (1年2月3天) |

```java
Duration d = Duration.between(start, end);
Period p = Period.between(startDate, endDate);

d.toDays();    // 总天数
p.getMonths();  // 月份部分
```
