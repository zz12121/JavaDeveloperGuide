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
```

### 拼接与其他

```java
String.join("-", "2026", "04", "18");  // "2026-04-18"（JDK 8+）
"a".repeat(3);                          // "aaa"（JDK 11+）
"  ".isBlank();                         // true（JDK 11+）
```

## 关联知识点

