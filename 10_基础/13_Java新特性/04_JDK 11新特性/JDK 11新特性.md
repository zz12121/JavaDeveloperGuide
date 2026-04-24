---
title: JDK 11新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# String 改进

## 核心结论

JDK 11 对 `String` 类新增了多个实用方法：`isBlank()`、`lines()`、`strip()`/`stripLeading()`/`stripTrailing()`、`repeat()`、`formatted()`（JDK 15）等，简化了常见的字符串处理操作。JDK 12 新增了 `indent()`、`transform()` 等方法。这些方法大多是对已有常见操作的标准化封装。

---

## 深度解析

### 1. isBlank() — 判断空白（JDK 11）

```java
"  ".isBlank();       // true（空格也是空白）
"\t\n".isBlank();     // true（制表符、换行符）
"".isBlank();         // true
"hello".isBlank();    // false

// 对比 isEmpty()
"  ".isEmpty();       // false（有内容，只是空格）
"".isEmpty();         // true
```

### 2. strip() 系列 — 去除空白（JDK 11）

```java
String s = "  hello  ";

s.strip();            // "hello"（去除首尾 Unicode 空白，包括 \u2003 等）
s.stripLeading();     // "hello  "（去除开头空白）
s.stripTrailing();    // "  hello"（去除末尾空白）

// 与 trim() 的区别
String s2 = "\u2003hello\u2003";  // \u2003 是全角空格（Em Space）
s2.trim();            // "\u2003hello\u2003"（trim 不识别 Unicode 空白）
s2.strip();           // "hello"（strip 识别所有 Unicode 空白字符）
```

> `strip()` 基于 `Character.isWhitespace()`，`trim()` 只去除 `char <= ' '` 的字符。

### 3. lines() — 按行分割（JDK 11）

```java
String multiline = "line1\nline2\r\nline3";

List<String> lines = multiline.lines().collect(Collectors.toList());
// ["line1", "line2", "line3"]

// 对比 split()
multiline.split("\\R");  // 需要正则，lines() 更简洁
```

### 4. repeat() — 重复字符串（JDK 11）

```java
"abc".repeat(3);   // "abcabcabc"
"*".repeat(5);     // "*****"
" ".repeat(4);     // "    "
```

### 5. formatted() — 格式化（JDK 15）

```java
// JDK 15 之前
String.format("Hello, %s!", name);

// JDK 15 之后
"Hello, %s!".formatted(name);
```

### 6. 其他实用方法

```java
// transform() — 链式转换（JDK 12）
"hello".transform(s -> s.toUpperCase())
       .transform(s -> ">> " + s + " <<");
// ">> HELLO <<"

// indent() — 缩进（JDK 12）
"line1\nline2".indent(4);
// "    line1\n    line2\n"

// String.compareToIgnoreCase 的常量形式
String s = "hello";
s.equalsIgnoreCase("HELLO");  // true

// describeConstable() — 返回 Optional<String>（JDK 12，配合 Constable 接口）
"hello".describeConstable();  // Optional[hello]
```

### 7. JDK 8~21 String 方法速查

| JDK | 方法 | 说明 |
|-----|------|------|
| 8 | `join()` | 静态方法，拼接字符串 |
| 11 | `isBlank()` | 判断是否为空白 |
| 11 | `strip()` / `stripLeading()` / `stripTrailing()` | 去除 Unicode 空白 |
| 11 | `lines()` | 按行分割为 Stream |
| 11 | `repeat(int)` | 重复 n 次 |
| 12 | `indent(int)` | 缩进调整 |
| 12 | `transform(Function)` | 链式转换 |
| 15 | `formatted(Object...)` | 格式化字符串 |
| 15 | `stripIndent()` | 去除缩进 |

---

## 代码示例

```java
// 实际场景：处理用户输入
String input = "  \n  hello\n  \n  world\n  ";

// JDK 11+
input.lines()
     .map(String::strip)
     .filter(s -> !s.isBlank())
     .collect(Collectors.joining(", "));
// "hello, world"

// 生成 CSV 行
String csv = String.join(",", List.of("name", "age", "city"));
// "name,age,city"
```

---

## 关联知识点

