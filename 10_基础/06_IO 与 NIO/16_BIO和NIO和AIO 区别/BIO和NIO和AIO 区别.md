---
title: BIO / NIO / AIO 区别
tags:
  - Java/IO
  - 对比型
module: 06_IO与NIO
created: 2026-04-18
---

# BIO / NIO / AIO 区别

## 三种 IO 模型

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| 全称 | Blocking IO | Non-blocking IO | Asynchronous IO |
| 模型 | **同步阻塞** | **同步非阻塞** | **异步非阻塞** |
| JDK 版本 | 1.0 | 1.4 | 1.7 |
| 操作方式 | 流（Stream） | 缓冲区（Buffer）+ 通道（Channel） | 回调 / Future |
| 线程模型 | 一连接一线程 | Selector 多路复用 | 回调驱动 |

## 详细对比

### BIO（同步阻塞）

```
客户端A ──→ 线程1（阻塞等待数据）
客户端B ──→ 线程2（阻塞等待数据）
客户端C ──→ 线程3（阻塞等待数据）
```

- 读写时线程**阻塞**，直到数据准备好
- **一连接一线程**，高并发时线程数爆炸
- 编程简单，适合低并发场景

### NIO（同步非阻塞）

```
Selector（1个线程）
  ├── Channel A（就绪 → 处理）
  ├── Channel B（未就绪 → 跳过）
  └── Channel C（就绪 → 处理）
```

- 线程查询 Channel 是否就绪，未就绪**立即返回**（不阻塞）
- 通过 Selector 实现**多路复用**，少量线程管理大量连接
- 适合高并发场景（Netty 基于 NIO）

### AIO（异步非阻塞）

```
线程发起 read 请求 → 立即返回 → 继续做其他事
                              ↓ 数据准备好
                         回调方法 / Future
```

- 发起 I/O 操作后**立即返回**，数据准备好后通过**回调**或 **Future** 通知
- 真正的异步，I/O 操作由 OS 完成
- 适合需要极致性能的场景

## 同步 vs 异步

| 维度 | 同步 | 异步 |
|------|------|------|
| 发起者 | 用户线程**自己**等待结果 | OS 完成后**通知**用户线程 |
| BIO | 同步阻塞 | — |
| NIO | 同步非阻塞（线程轮询） | — |
| AIO | — | 异步非阻塞 |

> NIO 虽然是非阻塞，但仍然是**同步**的——线程需要主动轮询 Selector 获取就绪事件。AIO 是真正的**异步**——线程发起操作后不关心结果，等回调通知。

## AIO 示例

```java
// AsynchronousFileChannel（异步文件读写）
AsynchronousFileChannel channel = AsynchronousFileChannel.open(
    Paths.get("data.txt"), StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);

// 方式一：Future
Future<Integer> result = channel.read(buffer, 0);
while (!result.isDone()) {
    // 做其他事情
}
int bytesRead = result.get();

// 方式二：CompletionHandler 回调
channel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        System.out.println("读取完成: " + result + " 字节");
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        exc.printStackTrace();
    }
});
```

## 应用场景

| 场景 | 推荐 | 说明 |
|------|------|------|
| 低并发文件操作 | BIO | 简单直接 |
| 高并发网络通信 | **NIO**（Netty） | Selector 多路复用 |
| 高性能文件操作 | AIO | 异步回调 |
| 数据库连接 | BIO | 连接池本身做了优化 |

## 关联知识点
