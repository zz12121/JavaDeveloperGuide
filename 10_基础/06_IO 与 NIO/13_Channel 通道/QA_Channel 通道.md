---
title: Channel 通道面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
updated: 2026-04-25
---

# Channel 通道

## Q1：Channel 和 Stream（流）的区别？

**A：**

| 维度 | Stream（IO 流） | Channel（NIO 通道） |
|------|----------------|-------------------|
| 方向 | 单向（输入流/输出流分开） | 双向（可读可写） |
| 阻塞性 | 只能阻塞式操作 | 可设置为非阻塞 |
| 配合使用 | 直接读写数据 | 必须配合 Buffer 使用 |
| 异步 | 不支持 | 支持异步操作（AIO） |
| 典型实现 | FileInputStream / BufferedReader | FileChannel / SocketChannel |

简而言之：**Stream 面向字节/字符，Channel 面向缓冲区**。

---

## Q2：Channel 有哪些主要类型？

**A：**

| 类型 | 说明 | 阻塞/非阻塞 |
|------|------|:---------:|
| FileChannel | 文件读写通道 | 只能阻塞 |
| DatagramChannel | UDP 数据报通道 | 可设置 |
| SocketChannel | TCP 客户端通道 | 可设置 |
| ServerSocketChannel | TCP 服务端监听通道 | 可设置 |

其中 **FileChannel 不能设置为非阻塞**，因为文件 I/O 不存在"就绪"概念——文件总是可读可写的。

---

## Q3：FileChannel 为什么不能设置为非阻塞？

**A：**

文件 I/O 和网络 I/O 本质不同：

- **网络 I/O**：数据从网卡到达，存在"数据还没到"的等待阶段——这就是"就绪"的由来，非阻塞才有意义
- **文件 I/O**：磁盘操作由 OS 统一管理，不存在"文件数据就绪"的概念，文件内容总是"可读"的

因此 `FileChannel.configureBlocking(false)` **会抛出异常**：
```
java.nio.channels.IllegalChannelModeException:
  FileChannel cannot be switched into non-blocking mode
```

如果需要在文件 I/O 中实现非阻塞效果，可以使用 **AIO（AsynchronousFileChannel）**。

---

## Q4：transferTo() 和 transferFrom() 有什么优势？

**A：**

```java
// 传统方式：read + write（4次拷贝）
FileInputStream fis = new FileInputStream("source.txt");
FileOutputStream fos = new FileOutputStream("dest.txt");
byte[] buf = new byte[8192];
int len;
while ((len = fis.read(buf)) != -1) {
    fos.write(buf, 0, len);
}

// transferTo（零拷贝，2次拷贝）
FileChannel in = new FileInputStream("source.txt").getChannel();
FileChannel out = new FileOutputStream("dest.txt").getChannel();
long transferred = in.transferTo(0, in.size(), out);
```

`transferTo()` 利用 OS 的 **零拷贝技术（sendfile）**，数据直接在内核态从源文件传输到目标 Socket/文件，**不经过用户态内存**，大幅减少上下文切换和内存拷贝次数。

Linux 中一次 `transferTo` 最多传输 2GB 数据，超过则需要循环调用。

---

## Q5：Channel 的 scatter/gather 是什么？

**A：**

- **Scatter（分散读取）**：将一个 Channel 的数据读取到**多个 Buffer** 中
- **Gather（聚集写入）**：将**多个 Buffer** 的数据写入到**一个 Channel** 中

```java
// Scatter — 适合协议解析（头部+body分开存储）
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);
ByteBuffer[] buffers = { header, body };
channel.read(buffers);  // 数据依次填满 header 再填 body

// Gather — 适合拼包发送
ByteBuffer buf1 = ByteBuffer.wrap("Header".getBytes());
ByteBuffer buf2 = ByteBuffer.wrap("Body".getBytes());
channel.write(new ByteBuffer[]{ buf1, buf2 });
```

这在处理固定格式协议（如 HTTP 协议头和消息体分离）时非常有用。

---

## Q6：如何获取一个 Channel？

**A：**

```java
// 1. 从已有的 IO 流/通道获取
FileInputStream fis = new FileInputStream("file.txt");
FileChannel fc1 = fis.getChannel();

// 2. 通过 RandomAccessFile
RandomAccessFile raf = new RandomAccessFile("file.txt", "rw");
FileChannel fc2 = raf.getChannel();

// 3. 通过 FileChannel.open()（JDK 7+ 推荐）
FileChannel fc3 = FileChannel.open(Paths.get("file.txt"),
    StandardOpenOption.READ, StandardOpenOption.WRITE);

// 4. Socket 通道
ServerSocketChannel ssc = ServerSocketChannel.open();
SocketChannel sc = ssc.accept();
```
