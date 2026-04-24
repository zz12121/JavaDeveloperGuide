---
title: JDK 11新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# String 改进

## Q1：strip() 和 trim() 有什么区别？

**A**：
- `trim()`：只去除 ASCII 码小于等于空格（`' '`，即 `\u0020`）的空白字符
- `strip()`：基于 `Character.isWhitespace()`，能识别所有 Unicode 空白字符（包括全角空格 `\u2003`、不间断空格 `\u00A0` 等）
```java
String s = "\u2003hello\u2003";
s.trim();   // "\u2003hello\u2003"（无法去除）
s.strip();  // "hello"（正常去除）
```

---

## Q2：isBlank() 和 isEmpty() 有什么区别？

**A**：
- `isEmpty()`：长度为 0 时返回 true（`str.length() == 0`）
- `isBlank()`：只包含空白字符或长度为 0 时返回 true
```java
"  ".isEmpty();    // false（有内容）
"  ".isBlank();   // true（只有空白）
"".isEmpty();      // true
"".isBlank();     // true
```

---

## Q3：lines() 和 split("\\R") 有什么区别？

**A**：功能相似，但 `lines()` 有优势：
- `lines()` 返回 `Stream<String>`，可以直接链式处理
- `lines()` 不会返回末尾的空字符串（trailing empty line）
- `split("\\R")` 返回数组，且可能在末尾包含空字符串
```java
"a\nb\n".lines().collect(Collectors.toList());  // ["a", "b"]
"a\nb\n".split("\\R");                          // ["a", "b"]
```

---

## Q4：repeat() 有什么实际用途？

**A**：常见用途包括生成分隔线、缩进、填充等：
```java
"-".repeat(50);           // 生成 50 个减号的分隔线
" ".repeat(4);            // 生成 4 个空格的缩进
"0".repeat(8 - hash.length()); // 哈希值前补零
```
