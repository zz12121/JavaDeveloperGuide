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

## Q：split() 有哪些坑？

**A：** split() 有两个主要坑点：

**1. 尾部空字符串被丢弃（JDK 8）**：
```java
"a,b,".split(",");        // JDK 8: ["a", "b"]（尾部空串被丢弃）
"a,b,".split(",", -1);    // JDK 11+: ["a", "b", ""]（-1 保留尾部）
"".split(",");             // []（不是 [""]）
```

**2. 正则特殊字符**：`. | + *` 等是正则元字符，必须转义：
```java
"a.b".split(".");         // []（"." 匹配任意字符，整个串被匹配完）
"a.b".split("\\.");       // ["a", "b"]（转义后按字面量处理）
```

## Q：为什么说 substring() 在 JDK 6 可能导致内存泄漏？

**A：** JDK 6 中 substring() 的实现如下：

```java
// JDK 6 源码（简化）
class String {
    private int offset;  // 共享原数组的起始位置
    private int count;
    private char[] value;

    public String substring(int begin, int end) {
        return new String(offset + begin, end - begin, value);
    }
}
```

substring 创建的新 String 共享原字符串的 char[] 数组。当原字符串很大（如从文件读取了几MB内容），而只取其中一小部分时：

```java
String big = readLargeFile();  // 10MB
String small = big.substring(0, 10);  // 只用了10个字符
// small 和 big 共享同一个 10MB 的 char[]，导致整个 10MB 无法被 GC
```

JDK 7+ 修复了这个问题：`substring()` 会 `new String(Arrays.copyOfRange(...))`，创建独立的副本。**实际开发中应避免在大字符串上频繁调用 substring()，即使 JDK 7+ 不会泄漏，也会频繁分配内存。**
