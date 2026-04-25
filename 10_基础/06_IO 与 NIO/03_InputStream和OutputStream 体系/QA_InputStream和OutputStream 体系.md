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

---

## Q4：ByteArrayInputStream/ByteArrayOutputStream 的特点和典型使用场景？

**A：**
`ByteArrayInputStream` 和 `ByteArrayOutputStream` 是基于内存字节数组的流，不需要关闭：

**典型场景：**
```java
// 1. 内存中构建数据，然后转为字节数组
ByteArrayOutputStream baos = new ByteArrayOutputStream();
try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
    oos.writeObject(myObject);
}
byte[] data = baos.toByteArray();  // 获取序列化后的字节

// 2. 将字节数组作为输入流处理
byte[] xmlData = getXmlBytes();
try (InputStream is = new ByteArrayInputStream(xmlData)) {
    // 复用同一个字节数组
}
```

**特点：**
- 无需关闭（`close()` 是空操作）
- `ByteArrayOutputStream` 自动扩容（默认 32 字节，按 2 倍扩容）
- 适合"内存中转"场景，如序列化/反序列化

---

## Q5：FilterInputStream 在 IO 体系中的角色是什么？

**A：**
`FilterInputStream` 是**装饰器模式的基类**，本身不实现具体的读写功能，而是**包装另一个 InputStream**，在读写过程中添加额外功能：

```java
// FilterInputStream 源码（简化）
public class FilterInputStream extends InputStream {
    protected volatile InputStream in;  // 被包装的流
    
    public FilterInputStream(InputStream in) {
        this.in = in;
    }
    
    public int read() throws IOException {
        return in.read();  // 默认委托给被包装的流
    }
}
```

**常见子类：**
| 类 | 功能 |
|---|---|
| `BufferedInputStream` | 添加缓冲，减少系统调用 |
| `DataInputStream` | 读取 Java 基本数据类型 |
| `PushbackInputStream` | 把数据推回流中回头再读 |

**装饰器模式优势：** 可以在运行时**动态组合**多个功能：
```java
InputStream in = new BufferedInputStream(           // 缓冲
                new DataInputStream(                 // 基本类型
                    new FileInputStream("data.bin") // 文件
                ));
```

---

## Q6：read() 返回 int 而不是 byte 有什么讲究？

**A：**
这是 Java IO 设计的一个小技巧：

```java
public abstract int read() throws IOException;

// 返回：0~255 表示读取到的字节，-1 表示流结束
```

**为什么用 int 而不是 byte？**

因为 `-1` 需要用来表示"流结束"：
- `byte` 的范围是 -128 ~ 127，无法同时表示 256 个字节值和"结束"状态
- `int` 的范围足够大，先读取为 `int`，如果是 -1 就表示结束，否则强制转型为 `byte`

```java
int b = in.read();
if (b == -1) {
    // 流结束
} else {
    byte actual = (byte) b;  // 转回 byte
}
```

**面试延伸：** `InputStream` 的 `read(byte[])` 也返回 `int` 而非 `byte[]` 长度，因为可能返回 -1。

