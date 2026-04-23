---
title: InputStream / OutputStream 体系
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# InputStream / OutputStream 体系

## InputStream（字节输入流）

所有字节输入流的抽象基类，核心方法：

```java
public abstract int read()                     // 读取一个字节，返回 -1 表示结束
public int read(byte b[])                      // 读取到字节数组
public int read(byte b[], int off, int len)    // 读取指定长度到数组
public long skip(long n)                       // 跳过 n 字节
public int available()                         // 可读字节数
public void close()                            // 关闭流
```

### 常用实现

| 类 | 说明 | 使用场景 |
|----|------|---------|
| `FileInputStream` | 从文件读取字节 | 读取任意文件 |
| `ByteArrayInputStream` | 从字节数组读取 | 内存中处理字节数据 |
| `BufferedInputStream` | 带缓冲的字节输入 | 提高读取性能 |
| `DataInputStream` | 读取基本数据类型 | 读取 int、double 等 |
| `ObjectInputStream` | 反序列化对象 | 恢复 Java 对象 |
| `PushbackInputStream` | 支持回推字节 | 编译器词法分析 |

### 使用示例

```java
// 基本读取（每次一个字节，效率低）
try (InputStream is = new FileInputStream("data.bin")) {
    int b;
    while ((b = is.read()) != -1) {
        // 处理字节
    }
}

// 缓冲读取（推荐）
try (BufferedInputStream bis = new BufferedInputStream(
        new FileInputStream("data.bin"))) {
    byte[] buffer = new byte[1024];
    int len;
    while ((len = bis.read(buffer)) != -1) {
        // 处理 buffer[0..len-1]
    }
}
```

## OutputStream（字节输出流）

所有字节输出流的抽象基类，核心方法：

```java
public abstract void write(int b)                   // 写入一个字节
public void write(byte b[])                         // 写入整个字节数组
public void write(byte b[], int off, int len)       // 写入指定长度
public void flush()                                 // 刷新缓冲区
public void close()                                 // 关闭流（自动 flush）
```

### 常用实现

| 类 | 说明 | 使用场景 |
|----|------|---------|
| `FileOutputStream` | 写入字节到文件 | 写入任意文件 |
| `ByteArrayOutputStream` | 写入到字节数组 | 内存中构建字节数据 |
| `BufferedOutputStream` | 带缓冲的字节输出 | 提高写入性能 |
| `DataOutputStream` | 写入基本数据类型 | 写入 int、double 等 |
| `ObjectOutputStream` | 序列化对象 | 存储 Java 对象 |
| `PrintStream` | 格式化输出（`System.out`） | 控制台输出、日志 |

### 使用示例

```java
// 基本写入
try (OutputStream os = new FileOutputStream("output.bin")) {
    os.write("Hello".getBytes());
}

// 缓冲写入
try (BufferedOutputStream bos = new BufferedOutputStream(
        new FileOutputStream("output.bin"))) {
    byte[] data = "Hello World".getBytes();
    bos.write(data);
    bos.flush();  // 手动刷新（close 也会自动 flush）
}
```

## 文件复制示例

```java
// 字节流复制文件（通用，支持任意文件类型）
try (InputStream is = new FileInputStream("src.bin");
     OutputStream os = new FileOutputStream("dest.bin")) {
    byte[] buffer = new byte[8192];
    int len;
    while ((len = is.read(buffer)) != -1) {
        os.write(buffer, 0, len);
    }
}
```

## 关联知识点
