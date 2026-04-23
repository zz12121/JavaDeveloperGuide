---
title: Selector 选择器
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# Selector 选择器

## Q1：Selector 的作用和工作原理？

**A：**
Selector 实现 I/O 多路复用，一个线程管理多个 Channel。Channel 注册到 Selector 后，Selector 监听这些 Channel 上的事件（连接、读、写），调用 `select()` 阻塞等待，有事件就绪后返回，开发者遍历处理。底层依赖 OS 的 epoll/kqueue。

## Q2：Selector 的 select() 方法有几种？

**A：**
- `select()`：阻塞直到有事件就绪
- `select(timeout)`：阻塞最多 timeout 毫秒
- `selectNow()`：不阻塞，立即返回

## Q3：处理 selectedKeys 时为什么要手动 remove？

**A：**
Selector 不会自动移除已处理的 Key。下次 `select()` 返回时，未 remove 的 Key 仍然在集合中，导致重复处理。

## Q4：Selector 底层依赖什么？

**A：**
- Linux：**epoll**
- macOS：kqueue
- Windows：select

Netty 在 Linux 上默认使用 epoll，性能最优。
