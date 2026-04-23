---
title: Reader / Writer 体系
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# Reader / Writer 体系

## Reader（字符输入流）

所有字符输入流的抽象基类，核心方法：

```java
public int read()                          // 读取一个字符，返回 -1 表示结束
public int read(char cbuf[])               // 读取到字符数组
public int read(char cbuf[], int off, int len) // 读取指定长度
public long skip(long n)                   // 跳过字符
public void close()
```

### 常用实现

| 类 | 说明 | 使用场景 |
|----|------|---------|
| `FileReader` | 从文件读取字符 | 读取文本文件 |
| `StringReader` | 从字符串读取 | 测试、字符串处理 |
| `BufferedReader` | 带缓冲，支持 `readLine()` | **最常用**，按行读取 |
| `InputStreamReader` | 字节流→字符流桥梁 | 指定编码读取 |

### 使用示例

```java
// FileReader + BufferedReader（最常用组合）
try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

// 指定编码读取
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("data.txt"), "UTF-8"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

## Writer（字符输出流）

所有字符输出流的抽象基类，核心方法：

```java
public void write(int c)                           // 写入一个字符
public void write(char cbuf[])                     // 写入字符数组
public void write(String str)                      // 写入字符串
public void write(String str, int off, int len)    // 写入字符串的一部分
public void flush()
public void close()
```

### 常用实现

| 类 | 说明 | 使用场景 |
|----|------|---------|
| `FileWriter` | 写入字符到文件 | 写入文本文件 |
| `StringWriter` | 写入到字符串 | 构建字符串 |
| `BufferedWriter` | 带缓冲，支持 `newLine()` | **最常用**，提高写入性能 |
| `OutputStreamWriter` | 字符流→字节流桥梁 | 指定编码写入 |
| `PrintWriter` | 格式化输出 | 写日志、格式化文本 |

### 使用示例

```java
// FileWriter + BufferedWriter
try (BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    bw.write("Hello");
    bw.newLine();
    bw.write("World");
}

// 指定编码写入
try (BufferedWriter bw = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream("output.txt"), "UTF-8"))) {
    bw.write("中文内容");
}
```

## 字符流 vs 字节流的底层关系

```
FileReader           ── 底层 → FileInputStream + InputStreamReader(默认编码)
BufferedReader       ── 包装 → 任何 Reader
InputStreamReader    ── 桥梁 → InputStream + Charset
FileWriter           ── 底层 → FileOutputStream + OutputStreamWriter(默认编码)
BufferedWriter       ── 包装 → 任何 Writer
OutputStreamWriter   ── 桥梁 → OutputStream + Charset
```

## 注意事项

1. **字符流内部有缓冲**，不调用 `flush()` 或 `close()` 可能数据写不进去
2. `FileReader`/`FileWriter` 使用系统默认编码，不同平台可能不同，**推荐指定编码**
3. `BufferedReader.readLine()` 不包含行尾换行符

## 关联知识点
