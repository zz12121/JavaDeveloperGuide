---
title: String 常用方法
tags:
  - Java/字符串
  - 原理型
module: 09_字符串
created: 2026-04-18
---

# String 常用方法

## 核心结论

String 提供了丰富的操作方法，常用方法可分为：判断类（equals/isEmpty/contains）、获取类（length/charAt/substring/indexOf）、转换类（toUpperCase/trim/toCharArray）、替换类（replace/replaceAll/split）。

## 深度解析

### 判断类方法

```java
String s = "Hello World";

s.equals("Hello World");          // true（内容比较，区分大小写）
s.equalsIgnoreCase("hello world"); // true（忽略大小写）
s.isEmpty();                       // false（长度为 0 返回 true）
s.isBlank();                       // false（JDK 11+，空白字符也返回 true）
s.contains("World");               // true
s.startsWith("Hello");             // true
s.endsWith("World");               // true
s.matches("H.*d");                 // true（正则匹配）
```

### 获取类方法

```java
String s = "Hello World";

s.length();            // 11
s.charAt(0);           // 'H'
s.indexOf("World");    // 6（首次出现位置，没有返回 -1）
s.indexOf("o", 5);    // 7（从位置 5 开始查找）
s.lastIndexOf("o");    // 7（最后出现位置）
s.substring(6);        // "World"（从 6 到末尾）
s.substring(0, 5);     // "Hello"（左闭右开 [0, 5)）
```

### 转换类方法

```java
String s = "  Hello World  ";

s.trim();                      // "Hello World"（去除首尾空白）
s.strip();                     // "Hello World"（JDK 11+，支持 Unicode 空白）
s.stripLeading();              // "Hello World  "（去除前导空白）
s.stripTrailing();             // "  Hello World"（去除尾部空白）
s.toUpperCase();               // "  HELLO WORLD  "
s.toLowerCase();               // "  hello world  "
s.toCharArray();               // char[] 数组
s.getBytes();                  // byte[] 数组（默认编码）
s.getBytes("UTF-8");           // byte[] 数组（指定编码）
String.valueOf(123);           // "123"（基本类型转 String）
```

### 替换与分割方法

```java
String s = "Hello World";

s.replace("World", "Java");    // "Hello Java"（替换所有匹配字符序列）
s.replaceAll("o", "0");        // "Hell0 W0rld"（正则替换）
s.replaceFirst("o", "0");      // "Hell0 World"（正则替换第一个）
s.split(" ");                   // ["Hello", "World"]（按分隔符拆分）
s.split(" ", 2);               // ["Hello", "World"]（限制分割次数）

### 常见陷阱

#### 1. split() 忽略尾部空字符串（JDK 8 及之前）

```java
// JDK 8：尾部空字符串被丢弃
"a,b,c,".split(",");        // ["a", "b", "c"]（3个，最后一个被丢弃）
"a,,b".split(",");          // ["a", "b"]（中间连续逗号产生空串，保留）
"".split(",");               // []（空字符串返回空数组，不是 [""]）
",".split(",");              // []（不是 ["", ""]）

// JDK 11+：可加第二个参数 -1 保留尾部空字符串
",".split(",", -1);         // ["", ""]
"a,b,c,".split(",", -1);    // ["a", "b", "c", ""]
```

#### 2. split() 的正则特殊字符

```java
// . | + * 等是正则特殊字符，直接写会按正则处理
"a.b.c".split(".");         // []（"." 匹配任意字符，整个字符串被匹配完）
"a.b.c".split("\\.");        // ["a", "b", "c"]（转义后按字面量处理）
"a|b|c".split("|");         // ["a", "|", "b", "|", "c"]（"|" 是或运算符）
"a|b|c".split("\\|");       // ["a", "b", "c"]
```

#### 3. substring() 内存泄漏（JDK 6 的坑）

```java
// JDK 6：substring() 不创建新 char[]，而是共享原字符串的数组
String s = createLargeString();  // 假设 10MB 的字符串
String sub = s.substring(0, 1); // 只用了1个字符
// 问题：sub 持有 10MB char[] 的引用，导致整个 10MB 无法被 GC

// JDK 7+：substring() 创建新 char[]，解决了这个问题
String sub = s.substring(0, 1);  // JDK 7+ 创建新的 1 字符数组
// 但仍有性能问题：substring() 仍分配新数组，应避免在循环中频繁调用

// 正确做法：显式截断确保创建新数组
String sub = new String(s.substring(0, 1));  // 强制创建新对象
String sub = s.substring(0, 1).intern();    // 放入常量池复用
```

#### 4. trim() vs strip() 空白字符范围不同

```java
// trim()：只处理 ASCII 空白（空格 \t \n \r 等）
// strip()：处理所有 Unicode 空白字符（JDK 11+）

String s = "\u2002hello\u2002";  // \u2002 是 Unicode 窄空格
s.trim();                        // "\u2002hello\u2002"（不处理）
s.strip();                       // "hello"（正确处理）

// 推荐：优先使用 strip()
```

### 拼接与其他

```java
String.join("-", "2026", "04", "18");  // "2026-04-18"（JDK 8+）
"a".repeat(3);                          // "aaa"（JDK 11+）
"  ".isBlank();                         // true（JDK 11+）
```

## 关联知识点

