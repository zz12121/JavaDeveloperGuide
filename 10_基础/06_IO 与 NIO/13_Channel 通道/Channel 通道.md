---
title: Channel 通道（FileChannel / SocketChannel / ServerSocketChannel）
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# Channel 通道

## Channel 概述

Channel 是 NIO 中数据传输的**双向通道**（区别于 Stream 的单向）：

| 维度 | Stream（BIO） | Channel（NIO） |
|------|---------------|----------------|
| 方向 | 单向（InputStream/OutputStream） | **双向**（可读可写） |
| 模式 | 阻塞 | 可阻塞或非阻塞 |
| 操作方式 | 直接读写 | 通过 Buffer 读写 |

## 主要实现

| Channel | 说明 | 阻塞/非阻塞 |
|---------|------|------------|
| `FileChannel` | 文件读写 | **只能阻塞** |
| `SocketChannel` | TCP 客户端 | 可非阻塞 |
| `ServerSocketChannel` | TCP 服务端 | 可非阻塞 |
| `DatagramChannel` | UDP | 可非阻塞 |

## FileChannel

FileChannel 只能通过 `FileInputStream`/`FileOutputStream`/`RandomAccessFile` 获取，且**不能设置为非阻塞**。

### 文件读取

```java
try (FileChannel channel = new FileInputStream("data.txt").getChannel()) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    int bytesRead = channel.read(buffer);  // 读取到 buffer
    buffer.flip();
    // 处理数据...
}
```

### 文件写入

```java
try (FileChannel channel = new FileOutputStream("output.txt").getChannel()) {
    ByteBuffer buffer = ByteBuffer.wrap("Hello".getBytes());
    channel.write(buffer);
}
```

### 文件复制（transferTo）

```java
try (FileChannel src = new FileInputStream("src.txt").getChannel();
     FileChannel dest = new FileOutputStream("dest.txt").getChannel()) {
    src.transferTo(0, src.size(), dest);  // 零拷贝，高效
}
```

> `transferTo()` 底层利用 OS 的零拷贝技术（sendfile），数据不经过用户态，直接在内核态传输。

## SocketChannel

```java
// 客户端
SocketChannel channel = SocketChannel.open(new InetSocketAddress("localhost", 8080));
channel.configureBlocking(false);  // 非阻塞

// 写入
ByteBuffer buffer = ByteBuffer.wrap("Hello Server".getBytes());
channel.write(buffer);

// 读取
ByteBuffer readBuffer = ByteBuffer.allocate(1024);
int bytesRead = channel.read(readBuffer);  // 非阻塞，可能返回 0
```

## ServerSocketChannel

```java
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);

while (true) {
    SocketChannel client = server.accept();  // 非阻塞，无连接返回 null
    if (client != null) {
        // 处理连接
    }
}
```

> 实际开发中，`ServerSocketChannel` 通常配合 `Selector` 使用。

## Channel vs Stream

| 维度 | Stream | Channel |
|------|--------|---------|
| 方向 | 单向 | 双向 |
| 缓冲 | 自带缓冲（BufferedStream） | 必须配合 Buffer |
| 阻塞 | 阻塞 | 可选非阻塞 |
| 零拷贝 | 不支持 | `transferTo()` 支持 |

## 关联知识点