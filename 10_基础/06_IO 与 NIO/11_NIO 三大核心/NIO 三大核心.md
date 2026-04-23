---
title: NIO 三大核心（Channel / Buffer / Selector）
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# NIO 三大核心（Channel / Buffer / Selector）

## NIO 概述

NIO（New IO / Non-blocking IO）是 JDK 1.4 引入的，与 BIO 的区别：

| 维度 | BIO（传统 IO） | NIO |
|------|---------------|-----|
| 模型 | 阻塞 | 非阻塞 |
| 操作单位 | 流（Stream） | 缓冲区（Buffer）+ 通道（Channel） |
| 线程模型 | 一连接一线程 | 一个线程管理多个连接（Selector） |

## 三大核心组件

### 1. Channel（通道）

数据传输的通道，双向的（可读可写）：

| 类 | 说明 |
|----|------|
| `FileChannel` | 文件读写 |
| `SocketChannel` | TCP 客户端 |
| `ServerSocketChannel` | TCP 服务端 |
| `DatagramChannel` | UDP |

```java
// FileChannel 示例
try (FileChannel channel = new FileInputStream("data.txt").getChannel()) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    channel.read(buffer);  // 从通道读到缓冲区
}
```

### 2. Buffer（缓冲区）

NIO 中数据读写都在缓冲区中进行：

```java
// 分配缓冲区
ByteBuffer buffer = ByteBuffer.allocate(1024);      // 堆内存
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);  // 直接内存

// 写入数据到 buffer
buffer.put("Hello".getBytes());

// 切换为读模式
buffer.flip();

// 读取数据
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}
```

### 3. Selector（选择器）

实现**多路复用**，一个线程管理多个 Channel：

```
                          ┌→ Channel 1（可读）
Selector ← 注册 ← Channel 2（就绪）──┼→ Channel 2（可写）
（监听）              ← Channel 3 ──┼→ Channel 3（可连接）
                          └→ ...
```

```java
Selector selector = Selector.open();
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);  // 非阻塞
ssc.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    int readyCount = selector.select();  // 阻塞直到有事件
    Set<SelectionKey> readyKeys = selector.selectedKeys();
    for (SelectionKey key : readyKeys) {
        if (key.isAcceptable()) {
            // 处理连接事件
        } else if (key.isReadable()) {
            // 处理读事件
        }
    }
}
```

## 三者的关系

```
Channel ←→ Buffer ←→ 数据

Selector 管理 ←→ 多个 Channel（监听事件）
```

1. **Channel**：数据的通道，双向
2. **Buffer**：数据的容器，读写都在 Buffer 中进行
3. **Selector**：多路复用器，一个线程管理多个 Channel

## NIO vs BIO 典型场景

| 场景 | BIO | NIO |
|------|-----|-----|
| 文件读写 | ✅ 简单直接 | ✅ FileChannel 更灵活 |
| 网络通信 | ❌ 一连接一线程 | ✅ Selector 多路复用 |
| 高并发 | ❌ 线程数 = 连接数 | ✅ 少量线程管理大量连接 |

## 关联知识点