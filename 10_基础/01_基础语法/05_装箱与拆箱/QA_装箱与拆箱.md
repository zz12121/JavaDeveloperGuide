---
title: 装箱与拆箱
tags:
  - Java/基础语法
  - 原理型
  - 问答
module: 01_基础语法
created: 2026-04-18
---

# 装箱与拆箱

## Q1：什么是装箱和拆箱？

**A**：
- **装箱（Boxing）**：基本类型 → 包装类，底层调用 `Integer.valueOf()`
- **拆箱（Unboxing）**：包装类 → 基本类型，底层调用 `intValue()`
- Java 5 引入自动装箱/拆箱，是编译器语法糖

---

## Q2：Integer 缓存池（核心考点）？

**A**：
- `Integer.valueOf()` 对 -128~127 范围返回缓存对象，超出范围创建新对象
- 导致 == 比较在此范围内 true，超出 false
- `new Integer()` 无论值多少都创建新对象

---

## Q3：注意事项？

**A**： 
- 比较包装类**必须用 `equals()`**，不要用 ==
- 包装类为 null 时自动拆箱会抛 **NullPointerException**
- 频繁装箱拆箱影响性能，大量运算建议用基本类型

---

## Q4：Integer a = 127; Integer b = 127; a == b 是 true 还是 false？为什么？

**A**：true。`Integer a = 127` 触发自动装箱 `Integer.valueOf(127)`，127 在 -128~127 缓存范围内，返回同一个缓存对象，所以 == 比较地址为 true。

---

## Q5：Integer a = 128; Integer b = 128; a == b 呢？

**A**：false。128 超出缓存范围，`valueOf()` 每次创建新的 Integer 对象，两个不同对象的 == 为 false。应该用 `equals()` 比较。

---

## Q6：自动拆箱时空指针怎么理解？

**A**：`Integer x = null; int y = x;` 会抛 NullPointerException。
因为自动拆箱编译为 `x.intValue()`，x 为 null 时调用方法自然报空指针。
所以在用包装类时一定要注意判空。

---

## 关联知识点
