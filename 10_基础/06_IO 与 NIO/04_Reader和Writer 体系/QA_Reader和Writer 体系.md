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

---

## Q4：Reader 的 mark() 和 reset() 有什么用？哪些 Reader 支持？

**A：**
`mark()` 标记当前位置，`reset()` 回到标记位置重新读取。典型用于"试探性读取"场景：

```java
// 读取 XML 文件头
BufferedReader reader = new BufferedReader(new FileReader("data.xml"));

// 标记，允许回退 4 个字符
reader.mark(4);

// 先读取 3 个字符看看是不是 BOM
char[] head = new char[3];
reader.read(head);
String header = new String(head);

// 如果不是 UTF-8 BOM，就回退重读
if (!header.equals("\uFEFF")) {
    reader.reset();  // 回到标记位置
}

// 继续正常读取...
```

**支持 mark 的 Reader：**
| Reader | markSupported() | mark 容量 |
|--------|-----------------|-----------|
| `BufferedReader` | ✅ | 可自定义 |
| `CharArrayReader` | ✅ | 可自定义 |
| `StringReader` | ✅ | Integer.MAX_VALUE |
| `FileReader` | ❌ | - |

---

## Q5：为什么字符流读取中文可能出现乱码？如何解决？

**A：**
**乱码的根本原因：** 字节流和字符流使用的**编码不一致**。

**典型场景：**
```java
// 文件是 UTF-8 编码，但 FileReader 用系统默认编码（Windows 是 GBK）
FileReader reader = new FileReader("data.txt");  // ❌ 可能乱码
```

**解决方案：**
```java
// 方案1：显式指定编码
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("data.txt"), StandardCharsets.UTF_8))) {
    // ...
}

// 方案2：Java 7+ 推荐
try (BufferedReader reader = Files.newBufferedReader(
        Paths.get("data.txt"), StandardCharsets.UTF_8)) {
    // ...
}
```

**编码相关知识：**
| 编码 | 中文占比 | 字节/字符 |
|------|----------|-----------|
| ASCII | 0 | 1 |
| ISO-8859-1 | 0 | 1 |
| GBK | 100% | 2 |
| UTF-8 | 可变 | 1~4（中文 3） |
| UTF-16 | 可变 | 2~4（中文 2） |

---

## Q6：Writer 的 flush() 和 close() 有什么区别？

**A：**

| 方法 | 作用 | 缓冲区 |
|------|------|--------|
| `flush()` | 强制将缓冲区数据写入底层流 | **不清空**缓冲区 |
| `close()` | 先 flush，再关闭流，释放资源 | **清空**缓冲区 |

**使用场景：**
```java
// 日志写入：不需要关闭流，但要立即看到日志
Writer logWriter = new FileWriter("app.log");
while (running) {
    logWriter.write(getLogMessage());
    logWriter.flush();  // ✅ 立即写入磁盘
}

// 文件写入：用 try-with-resources 自动 close
try (Writer w = new BufferedWriter(new FileWriter("data.txt"))) {
    w.write("hello");
}  // ✅ close 时自动 flush

// 网络写入（PrintWriter）：需要 flush 发送
PrintWriter out = new PrintWriter(socket.getOutputStream());
out.print("response");
out.flush();  // ✅ 立即发送
```

**面试重点：** `close()` 后不能继续写入，会抛 `Stream closed` 异常。

---

## Q7：InputStreamReader 和 OutputStreamWriter 的内部工作机制？

**A：**
`InputStreamReader` 和 `OutputStreamWriter` 本质上是一个**桥接器**，将字节流转换为字符流：

```java
// InputStreamReader 内部结构
public class InputStreamReader extends Reader {
    private final StreamDecoder sd;  // 内部用 StreamDecoder 解码
    
    public InputStreamReader(InputStream in, Charset cs) {
        sd = StreamDecoder.forInputStreamReader(in, null, cs.name());
    }
    
    public int read(char[] cbuf, int offset, int length) throws IOException {
        return sd.read(cbuf, offset, length);
    }
}
```

**工作流程（读取）：**
```
文件(字节) → FileInputStream(字节) → InputStreamReader(字节→字符) → BufferedReader(缓冲)
                  ↓                        ↓
            读取原始字节              CharsetDecoder 解码
```

**Charset 解码过程：**
```java
// InputStreamReader 读取时实际发生的事
ByteBuffer bb = ByteBuffer.allocate(1024);
bb.put(fileBytes);
bb.flip();
CharBuffer cb = Charset.forName("UTF-8").decode(bb);  // 字节→字符
```

**OutputStreamWriter 同理**：`Writer` → `StreamEncoder` → `OutputStream`

---

## Q8：字符流和字节流如何选择？

**A：**

**选择字节流（InputStream/OutputStream）：**
- 二进制文件：图片、音视频、压缩包、PDF、可执行文件
- 不关心文本内容的传输
- 自定义协议或序列化

**选择字符流（Reader/Writer）：**
- 纯文本文件：`.txt`、`.csv`、`.log`、`.json`、`.xml`
- 需要按字符/行处理文本
- 网络通信中的文本协议（HTTP header、FTP 等）

**常见场景对照：**

| 场景 | 推荐方案 |
|------|----------|
| 读取文本文件 | `Files.readAllLines()` / `BufferedReader` |
| 写入文本文件 | `Files.write()` / `BufferedWriter` |
| 文件复制（二进制） | `BufferedInputStream` + `BufferedOutputStream` |
| 序列化对象 | `ObjectOutputStream` |
| 读取网络响应 | `BufferedReader`（文本）或 `BufferedInputStream`（二进制） |

**最佳实践：**
```java
// Java 7+ 推荐：Files 工具类
String content = Files.readString(Path.of("data.txt"), StandardCharsets.UTF_8);
Files.writeString(Path.of("out.txt"), content, StandardCharsets.UTF_8);

// 或显式用 BufferedReader/Writer
try (BufferedReader reader = Files.newBufferedReader(Path.of("data.txt"));
     BufferedWriter writer = Files.newBufferedWriter(Path.of("out.txt"))) {
    // ...
}
```

