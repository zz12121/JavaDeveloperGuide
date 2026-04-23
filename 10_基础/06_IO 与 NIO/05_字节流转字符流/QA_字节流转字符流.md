---
title: 字节流转字符流面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# 字节流转字符流

## Q1：InputStreamReader 和 OutputStreamWriter 的作用？

**A：**
它们是字节流和字符流之间的**桥梁**（也叫转换流）。
- `InputStreamReader`：将 `InputStream`（字节）按指定编码解码为 `Reader`（字符）
- `OutputStreamWriter`：将 `Writer`（字符）按指定编码编码为 `OutputStream`（字节）

---

## Q2：为什么不直接用 FileReader？

**A：**
`FileReader` 内部就是 `InputStreamReader` + `FileInputStream`，但**无法指定编码**，使用系统默认编码。Windows 默认 GBK，Linux 默认 UTF-8，跨平台可能乱码。
涉及中文或需要指定编码时，应使用：
```java
new InputStreamReader(new FileInputStream("file.txt"), "UTF-8")
```

---

## Q3：如何将 GBK 编码的文件转为 UTF-8？

**A：**
```java
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("gbk.txt"), "GBK"));
     BufferedWriter bw = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream("utf8.txt"), "UTF-8"))) {
    String line;
    while ((line = br.readLine()) != null) {
        bw.write(line);
        bw.newLine();
    }
}
```
