---
title: InputStream / OutputStream 体系面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# InputStream / OutputStream 体系

## Q1：InputStream 的核心方法有哪些？

**A：**
- `int read()`：读取一个字节，返回 -1 表示流结束
- `int read(byte[] b)`：读取到字节数组，返回实际读取的字节数
- `int read(byte[] b, int off, int len)`：读取指定长度
- `long skip(long n)`：跳过 n 字节
- `void close()`：关闭流

---

## Q2：为什么读取文件推荐用缓冲而不是逐字节读取？

**A：**
逐字节 `read()` 每次调用都有系统调用开销。用 `BufferedInputStream` 或 `byte[]` 缓冲区可以一次读取多个字节，减少系统调用次数，大幅提升性能。

---

## Q3：用字节流复制文件的代码怎么写？

**A：**
```java
try (InputStream is = new FileInputStream("src.bin");
     OutputStream os = new FileOutputStream("dest.bin")) {
    byte[] buffer = new byte[8192];
    int len;
    while ((len = is.read(buffer)) != -1) {
        os.write(buffer, 0, len);
    }
}
```
使用 try-with-resources 自动关闭流，`8192` 字节缓冲区平衡内存和性能。
