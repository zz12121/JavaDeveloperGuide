---
title: Channel 通道面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# Channel 通道

## Q1：Channel 和 Stream 的区别？

**A：**
- Channel 是**双向**的（可读可写），Stream 是单向的
- Channel 可以设置为**非阻塞**，Stream 只能阻塞
- Channel 必须配合 Buffer 使用，Stream 直接读写

## Q2：FileChannel 为什么不能设置为非阻塞？

**A：**
文件 I/O 不存在"就绪"的概念——文件总是可读可写的。非阻塞的意义在于网络 I/O 等待数据到达的场景，文件 I/O 不需要。

## Q3：FileChannel 的 transferTo() 有什么优势？

**A：**
`transferTo()` 利用 OS 的**零拷贝**技术（sendfile），数据直接在内核态从源文件传输到目标文件，不经过用户态内存，减少拷贝次数，性能远优于手动 read + write。
