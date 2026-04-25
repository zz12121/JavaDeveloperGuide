---
title: 字节流转字符流面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# 字节流转字符流

## Q1：InputStreamReader 和 OutputStreamWriter 的作用是什么？

**A：**
它们是**字节流和字符流之间的桥梁**：
- `InputStreamReader`：将字节输入流按指定编码转换为字符输入流（字节 → 字符）
- `OutputStreamWriter`：将字符输出流按指定编码转换为字节输出流（字符 → 字节）

工作原理：读取字节后使用 `Charset` 解码，接收字符后使用 `Charset` 编码。

---

## Q2：为什么不能直接用 FileReader 处理中文文件？

**A：**
`FileReader` 内部使用**系统默认编码**（Windows 上是 GBK，Linux 上是 UTF-8），不同系统上行为不一致：

```java
// FileReader 源码
public FileReader(String fileName) throws FileNotFoundException {
    super(new FileInputStream(fileName));  // 使用默认编码！
}

// 正确做法：显式指定编码
new InputStreamReader(new FileInputStream("data.txt"), StandardCharsets.UTF_8)
new OutputStreamWriter(new FileOutputStream("out.txt"), "UTF-8")
```

跨平台 Java 应用必须显式指定编码，否则在不同操作系统上会有乱码问题。

---

## Q3：编码转换（GBK → UTF-8）如何实现？

**A：**
使用 InputStreamReader 和 OutputStreamWriter 的组合：

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

---

## Q4：字符流为什么要区分 UTF-8 和 GBK？

**A：**
UTF-8 和 GBK 对中文的编码方式不同：
- **GBK**：中文占 2 字节（范围 0x8140~0xFEFE）
- **UTF-8**：中文占 3 字节（以 0xE开头）

同一文件用错误编码读取会导致乱码：
```java
// 读 UTF-8 文件用 GBK 解码 → 乱码
// 读 GBK 文件用 UTF-8 解码 → 乱码
```

必须确保**编码和解码使用同一字符集**。

---

## Q5：InputStreamReader 的缓冲区机制？

**A：**
`InputStreamReader` 内部有一个缓冲区（默认 8KB），减少对底层 InputStream 的读取次数：

```java
public class InputStreamReader {
    private final StreamDecoder sd;  // 包含缓冲区
}

// 建议配合 BufferedReader 使用（提供行级缓冲）
BufferedReader br = new BufferedReader(
    new InputStreamReader(new FileInputStream("file.txt"), "UTF-8")
);
```

配合 BufferedReader 后，可以直接使用 `readLine()` 按行读取，提高效率。
