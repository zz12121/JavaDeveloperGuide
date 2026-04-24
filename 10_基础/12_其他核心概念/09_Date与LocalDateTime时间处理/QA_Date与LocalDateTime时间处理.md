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
