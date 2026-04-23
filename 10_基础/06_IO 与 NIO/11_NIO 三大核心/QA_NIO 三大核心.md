---
title: NIO 三大核心面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# NIO 三大核心

## Q1：NIO 的三大核心组件？

**A：**
1. **Channel（通道）**：双向数据通道，类似流但可以双向读写
2. **Buffer（缓冲区）**：数据的容器，读写都在 Buffer 中进行
3. **Selector（选择器）**：多路复用器，一个线程管理多个 Channel

## Q2：NIO 和 BIO 的核心区别？

**A：**
- BIO：面向流，阻塞式，一连接一线程
- NIO：面向缓冲区和通道，非阻塞，一个线程通过 Selector 管理多个连接

## Q3：Selector 的工作原理？

**A：**
Selector 可以注册多个 Channel，并监听这些 Channel 上的事件（连接、读、写）。调用 `select()` 阻塞等待，直到有 Channel 准备好 I/O 操作，然后返回就绪的 Channel 集合进行处理。这就是**I/O 多路复用**。
