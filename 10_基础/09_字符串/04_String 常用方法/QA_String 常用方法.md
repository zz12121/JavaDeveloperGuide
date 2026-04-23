---
title: String 常用方法
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String 常用方法

## Q：String 有哪些常用方法？

**A：** 按功能分类：
**判断类**：`equals()`、`equalsIgnoreCase()`、`isEmpty()`、`contains()`、`startsWith()`、`endsWith()`
**获取类**：`length()`、`charAt()`、`indexOf()`、`lastIndexOf()`、`substring()`
**转换类**：`trim()`、`toUpperCase()`、`toLowerCase()`、`toCharArray()`、`String.valueOf()`
**替换分割**：`replace()`、`replaceAll()`、`replaceFirst()`、`split()`
**其他**：`String.join()`、`repeat()`（JDK 11+）、`isBlank()`（JDK 11+）

## Q：substring() 的行为在 JDK 6 和 JDK 7+ 有什么区别？

**A：**
- **JDK 6**：`substring()` 共享原字符串的 `char[]` 数组，只修改 offset 和 count。如果原字符串很大而 substring 很小，会导致内存泄漏（大数组无法被 GC）
- **JDK 7+**：`substring()` 创建新的 `char[]` 数组并复制数据，不再共享。修复了内存泄漏问题，但复制有一定开销

## Q：trim() 和 strip() 有什么区别？

**A：**
- `trim()`：只去除 ASCII 字符中的空白（空格、\t、\n、\r 等，编码 ≤ '\u0020'）
- `strip()`（JDK 11+）：去除所有 Unicode 空白字符，包括中文全角空格等
如果需要处理国际化字符串，使用 `strip()` 更安全。
