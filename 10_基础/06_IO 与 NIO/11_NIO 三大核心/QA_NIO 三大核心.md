---
title: NIO 三大核心面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
updated: 2026-04-25
---

# NIO 三大核心

## Q1：NIO 的三大核心组件是什么？它们各自的作用？

**A：**

| 组件 | 作用 | 类比 |
|------|------|------|
| **Channel（通道）** | 数据传输的双向通道，类似 Stream 但可读可写 | 水管 |
| **Buffer（缓冲区）** | 数据的容器，所有读写操作都在 Buffer 中进行 | 水库 |
| **Selector（选择器）** | 多路复用器，一个线程监听多个 Channel 的就绪状态 | 监控中心 |

三者的协作关系：**数据通过 Channel 流入/流出 Buffer，Selector 负责监听哪些 Channel 已经就绪**。

---

## Q2：NIO 和 BIO 的核心区别？

**A：**

| 维度 | BIO（同步阻塞） | NIO（同步非阻塞） |
|------|:---:|:---:|
| 模型 | 面向流 | 面向缓冲区和通道 |
| 线程模型 | 一连接一线程 | Selector 多路复用 |
| 阻塞方式 | 读写时线程阻塞 | 轮询就绪，不阻塞 |
| 吞吐量 | 低（线程开销大） | 高（单线程管理多连接） |
| JDK 版本 | 1.0 | 1.4 |
| 适用场景 | 低并发、简单业务 | 高并发（Netty、Redis） |

> **关键区别**：BIO 是"等数据准备好再读"，NIO 是"问 Channel 数据好了没，没好先干别的"。

---

## Q3：为什么说 NIO 仍然属于"同步"IO？

**A：**

NIO 虽然叫 Non-blocking IO，但它是**同步非阻塞**的：

- `select()` 调用本身是**阻塞**的（除非用 `selectNow()`）
- 线程需要**主动轮询** Selector 获取就绪的 Channel
- 数据读写仍是用户线程自己完成的（只是不等数据到来）

真正的**异步非阻塞**是 **AIO**（JDK 1.7+），由 OS 完成 I/O 后通过回调通知用户线程：

```
BIO:  线程 → 等待数据 → 读取 → 处理     （全程阻塞）
NIO:  线程 → select()阻塞 → 轮询就绪 → 读取 → 处理  （部分阻塞）
AIO:  线程 → 发起请求立即返回 → 做其他事 → 回调通知   （完全不阻塞）
```

---

## Q4：NIO 为什么能提升高并发场景的性能？

**A：**

以 10000 个连接为例：

- **BIO**：需要 10000 个线程，每个线程阻塞在 `read()` 上 → 线程栈内存消耗巨大（1线程 ≈ 1MB）
- **NIO**：只需要 1 个线程（或少量线程池），通过 Selector 监听所有连接 → 只处理已就绪的连接

性能提升来源：
1. **减少线程数量**：避免了线程创建、切换的开销
2. **减少阻塞等待**：单线程可以服务大量连接
3. **零拷贝支持**：FileChannel 的 `transferTo()` 可实现内核态直接传输

但需要注意：**单线程 Selector 模式下，如果有耗时的业务逻辑，会阻塞其他就绪事件**。实际生产中会用**线程池**处理业务逻辑。

---

## Q5：Buffer 在 NIO 中的角色是什么？

**A：**

Buffer 是 NIO 数据的**唯一入口和出口**：

```java
// NIO 读：Channel → Buffer → 程序
ByteBuffer buffer = ByteBuffer.allocate(1024);
channel.read(buffer);     // 数据从 Channel 读入 Buffer
buffer.flip();            // 切换为读模式
byte[] data = new byte[buffer.remaining()];
buffer.get(data);         // 从 Buffer 取出数据

// NIO 写：程序 → Buffer → Channel
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put("Hello".getBytes());
buffer.flip();
channel.write(buffer);   // 数据从 Buffer 写入 Channel
```

这种设计避免了直接操作阻塞的 Stream，让用户对读写有更精细的控制（position、limit、mark）。
