---
title: BIO / NIO / AIO 区别面试题
tags:
  - Java/IO
  - 对比型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# BIO / NIO / AIO 区别

## Q1：BIO、NIO、AIO 的区别？

**A：**

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| 模型 | 同步阻塞 | 同步非阻塞 | 异步非阻塞 |
| 线程 | 一连接一线程 | Selector 多路复用 | 回调 |
| 版本 | JDK 1.0 | JDK 1.4 | JDK 1.7 |
| 适用 | 低并发 | **高并发（Netty）** | 高性能文件 IO |

---

## Q2：NIO 是同步还是异步？

**A：**
NIO 是**同步非阻塞**。线程通过 Selector 主动轮询检查哪些 Channel 就绪，然后处理。线程仍然需要主动去"问"状态，而不是被动等待通知。真正的异步是 AIO。

---

## Q3：为什么 Netty 不用 AIO？

**A：**
1. Linux 上 AIO 实现不完善，底层可能还是用 epoll 模拟
2. Netty 的 NIO + 多 Reactor 线程模型已经足够高效
3. AIO 的编程复杂度更高，且回调链路容易出问题
4. AIO 主要在 Windows IOCP 上有真正优势，但服务器主要是 Linux
