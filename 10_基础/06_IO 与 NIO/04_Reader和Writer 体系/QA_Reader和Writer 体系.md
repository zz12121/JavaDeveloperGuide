---
title: Reader / Writer 体系面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# Reader / Writer 体系

## Q1：字符流和字节流有什么关系？

**A：**
字符流底层仍然基于字节流。`Reader` 的底层是 `InputStream` + 编码转换，`Writer` 的底层是 `OutputStream` + 编码转换。字符流是字节流的"便利封装"，自动处理字符编码问题。

---

## Q2：为什么 FileWriter 不推荐用来读取包含中文的文件？

**A：**
`FileReader` 使用系统默认编码（Windows 是 GBK，Linux 是 UTF-8），跨平台时可能乱码。推荐使用 `InputStreamReader` + `FileInputStream` 并**显式指定编码**：
```java
new InputStreamReader(new FileInputStream("data.txt"), "UTF-8")
```

---

## Q3：BufferedReader 的 readLine() 返回的字符串包含换行符吗？

**A：**
不包含。`readLine()` 返回一行内容，**不包含行尾的换行符**（`\n` 或 `\r\n`）。
