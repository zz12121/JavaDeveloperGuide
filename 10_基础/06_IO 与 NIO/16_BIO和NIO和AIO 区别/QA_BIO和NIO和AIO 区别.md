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

### 核心对比表

| **维度** | **BIO** | **NIO** | **AIO** |
|----------|---------|---------|---------|
| **全称** | Blocking I/O | Non-Blocking I/O (New I/O) | Asynchronous I/O |
| **模型** | 同步阻塞 | **同步非阻塞** | **异步非阻塞** |
| **JDK 版本** | JDK 1.0（传统 IO） | JDK 1.4+ | JDK 1.7+ (NIO.2) |
| **核心 API** | `InputStream/OutputStream` | `Channel` + `Buffer` + `Selector` | `AsynchronousSocketChannel` + `CompletionHandler` |
| **线程模型** | 一连接一线程 | Reactor（Selector 多路复用） | Proactor（操作系统回调） |
| **并发能力** | 低（受限于线程数） | 高（单线程处理万级连接） | 理论上最高 |
| **编程复杂度** | ⭐ 简单 | ⭐⭐⭐ 中等 | ⭐⭐ 较简单（但回调链复杂） |
| **适用场景** | 低并发、连接数少 | **高并发（Netty 基于此）** | 文件IO为主，网络IO Linux支持不完善 |
| **典型框架/应用** | 传统 Servlet (Tomcat BIO模式) | **Netty, MINA, Dubbo** | 异步文件操作 |

### 三种模型的通信流程图解

```
===== BIO：同步阻塞 =====
客户端                    服务端线程
  │                          │
  ├─connect()─────────────→ │ accept() ← 阻塞等待连接
  │                          ├─read()   ← 阻塞等待数据
  ├─send(data)────────────→ │          （数据到达前线程什么都做不了）
  │                          ├─process()
  │←──────────reply─────────┤ write()
  │                          └─(线程空闲等待下一个请求)
  │
  问题：10000个客户端 = 10000个线程 = 内存爆炸！


===== NIO：同步非阻塞 =====
客户端                    Selector(单线程)
  │                          │
  ├─register(OP_READ)────→  │ select() ← 阻塞等待任一事件
  ├─send(data)───────────→  │ ← 返回：channelX 可读
  │                          ├─channelX.read() → 非阻塞，立即返回
  │                          ├─channelY.read() → 非缓冲，立即返回
  │←──────────reply─────────┤ channelX.write() ← 非阻塞
  │
  特点：一个线程管理所有连接！
  "同步"=线程自己轮询检查事件；"非阻塞"=不会卡死在某一个连接上


===== AIO：异步非阻塞 =====
客户端                    操作系统内核              回调线程
  │                          │                        │
  ├─send(data)───────────→  │ 接收数据               │
  │                          │ 数据准备好后            │
  │                          ├─completed(结果)───────→│ callback.completed(result)
  │←──────────reply─────────┤                        │
  │
  特点：发起调用后完全不管了，操作系统做完后通知你
  "异步"=不需要主动去问，系统做好了会回调通知
```

### 代码对比：三种方式实现 Echo Server

```java
// ========== BIO 实现 ==========
public class BioEchoServer {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO服务器启动");
        while (true) {
            Socket client = serverSocket.accept();  // 阻塞：等待连接
            // 每个连接一个线程
            new Thread(() -> {
                try (Socket s = client;
                     InputStream in = s.getInputStream();
                     OutputStream out = s.getOutputStream()) {
                    byte[] buf = new byte[1024];
                    int len;
                    while ((len = in.read(buf)) != -1) {  // 阻塞：等待数据
                        out.write(buf, 0, len);            // 回显
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}


// ========== NIO 实现（简化版）==========
import java.nio.*;
import java.nio.channels.*;
import java.net.*;

public class NioEchoServer {
    public static void main(String[] args) throws Exception {
        Selector selector = Selector.open();
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.bind(new InetSocketAddress(8081));
        ssc.configureBlocking(false);
        ssc.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO服务器启动");
        while (true) {
            selector.select();  // 阻塞等待事件
            for (SelectionKey key : selector.selectedKeys()) {
                if (key.isAcceptable()) {
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    sc.register(selector, SelectionKey.OP_READ);
                }
                if (key.isReadable()) {
                    SocketChannel sc = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(256);
                    int readBytes = sc.read(buf);     // 非阻塞！立即返回
                    if (readBytes > 0) {
                        buf.flip();
                        sc.write(buf);                 // 回显
                    } else if (readBytes == -1) {
                        sc.close();                    // 客户端断开
                    }
                }
            }
            selector.selectedKeys().clear();
        }
    }
}


// ========== AIO 实现 ==========
import java.nio.channels.*;
import java.nio.ByteBuffer;
import java.net.*;

public class AioEchoServer {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server =
                AsynchronousServerSocketChannel.open();
        server.bind(new InetSocketAddress(8082));

        System.out.println("AIO服务器启动");

        // 异步接受连接 —— 不阻塞！通过回调处理
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
            @Override
            public void completed(AsynchronousSocketChannel client, Object attachment) {
                // 继续接受下一个连接（递归注册accept）
                server.accept(null, this);

                // 异步读 —— 读完后自动回调
                ByteBuffer buf = ByteBuffer.allocate(256);
                client.read(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buffer) {
                        if (result == -1) {
                            try { client.close(); } catch (Exception e) {}
                            return;
                        }
                        buffer.flip();
                        // 异步写 —— 写完后自动回调
                        client.write(buffer, null,
                            new CompletionHandler<Integer, Void>() {
                                @Override
                                public void completed(Integer written, Void att) {
                                    // 写完后再注册一次读取（循环）
                                    ByteBuffer newBuf = ByteBuffer.allocate(256);
                                    client.read(newBuf, newBuf, /*...*/);
                                }

                                @Override
                                public void failed(Throwable exc, Void att) {
                                    exc.printStackTrace();
                                }
                            });
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        exc.printStackTrace();
                    }
                });
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });

        // 主线程可以去做其他事...
        Thread.sleep(Long.MAX_VALUE);  // 保持进程运行
    }
}
```

---

## Q2：NIO 是同步还是异步？

**A：**

### 结论：NIO 是同步非阻塞（Synchronous Non-Blocking）

这是一个非常容易混淆的面试高频问题。

#### 关键区分：同步 vs 异步 的本质

```
判断标准是谁在负责数据的"搬运"工作：

同步（Synchronous）：应用程序（你的代码）亲自参与I/O操作的每个环节
  - 调用 read() 后，虽然不阻塞（立即返回），但你仍需自己去问"好了没？"
  - 数据从内核到用户空间的拷贝过程由应用线程完成

异步（Asynchronous）：应用程序只管发起请求，然后该干嘛干嘛
  - 整个I/O操作（包括内核处理和数据拷贝到用户空间）都由操作系统完成
  - 操作系统完成后通过信号或回调通知你"搞定了"
```

#### NIO 为什么是"同步"的？

```java
// NIO 的典型代码：
Selector selector = Selector.open();
channel.register(selector, OP_READ);

// 你的代码必须主动去"问"
while (true) {
    int ready = selector.select();      // ① 你主动询问："有事件吗？"
    Set<SelectionKey> keys = selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isReadable()) {
            channel.read(buffer);       // ② 你亲自执行读取（数据搬运）
            processData(buffer);         // ③ 你亲自处理数据
        }
    }
}
// 全程都是你的线程在干活！—— 这就是"同步"
// 只是它"非阻塞"—— 在①处不会一直等，没事件就先返回
```

#### AIO 为什么是真正的"异步"？

```java
// AIO 的典型代码：
AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();

ByteBuffer buffer = ByteBuffer.allocate(1024);

// 只管发起请求，然后走人！
channel.read(buffer, timeout, TimeUnit.MILLISECONDS, null,
    new CompletionHandler<Integer, Object>() {
        @Override
        public void completed(Integer result, Object attachment) {
            // 操作系统帮你读完数据后才调这个方法
            // 你甚至不知道是哪个线程在执行这里的代码
            buffer.flip();
            processData(buffer);  // 此时数据已经准备好了
        }
        @Override
        public void failed(Throwable exc, Object attachment) {
            exc.printStackTrace();
        }
});

// 发起 read 后主线程就可以去干别的了！
doOtherImportantWork();  // 不需要任何轮询
// 这才是真正的"异步"
```

### Richard Stevens 的经典分类法（POSIX 标准）

| **类型** | **同步/异步** | **阻塞/非阻塞** |
|----------|-------------|---------------|
| 同步阻塞 | 应用程序参与全程 | 等待 I/O 完成 |
| **同步非阻塞** | 应用程序参与全程 | 轮询/事件驱动 |
| 异步阻塞 | 极少见，几乎不用 | 等待 I/O 完成 |
| **异步非阻塞** | 操作系统代劳 | 通过回调/信号通知 |

> **Java NIO = 同步非阻塞（Synchronous Non-Blocking I/O）**
> **Java AIO = 异步非阻塞（Asynchronous Non-Blocking I/O）**
>
> 记忆口诀：**NIO 要自己 poll，AIO 让 OS call back。**

---

## Q3：为什么 Netty 不用 AIO？

**A：**

### 核心原因

尽管 AIO 看起来是最先进的 IO 模型，但 Netty（以及大多数高性能网络框架）选择基于 **NIO + Reactor 模式**构建，主要原因如下：

#### 原因1：Linux 上 AIO 的实现不成熟

```java
// Linux 上的 AIO (native aio / io_uring):
//
// - 早期 Linux kernel (< 5.1) 的 native AIO 只支持 Direct IO（绕过页缓存）
//   导致性能反而不如 epoll + 缓冲 IO
// - 即使后来的 io_uring 性能优秀，但也直到较新的内核版本才稳定
// - 而 epoll 从 Linux 2.5.44 (2003年) 就已成熟稳定
//
// Windows 上的 IOCP (Completion Port) 才是真正成熟的异步IO实现
// 但主流服务器操作系统是 Linux！
```

#### 原因2：NIO + Reactor 已经足够高效

```java
// Netty 基于 NIO 的 EpollEventLoop 实际性能：

// 单机测试数据（大致量级）：
// 连接数        QPS (requests/sec)    CPU使用率    延迟(p99)
// ─────────    ──────────────────    ─────────    ─────────
// 10,000        ~500,000             ~60%          < 1ms
// 100,000       ~450,000             ~70%          < 2ms
// 1,000,000     ~400,000             ~85%          < 5ms
//
// 这个性能水平对于绝大多数场景已经远超需求
// 引入 AIO 带来的复杂性不值得为了那一点性能提升而牺牲稳定性
```

#### 原因3：AIO 编程模型的问题

```java
// AIO 的回调地狱 (Callback Hell):

channel.read(buffer, null, new CompletionHandler<>() {
    @Override
    public void completed(Integer result, Attachment att) {
        parseHeader(buffer, header -> {
            channel.read(bodyBuffer, null, new CompletionHandler<>() {
                @Override
                public void completed(Integer bodyLen, Attachment att2) {
                    processRequest(header, body, response -> {
                        channel.write(response, null, new CompletionHandler<>() {
                            @Override
                            public void completed(Integer written, Attachment att3) {
                                // ... 还要再注册下一次读取
                            }
                            @Override
                            public void failed(Throwable t, Attachment att3) {
                                handleError(t);
                            }
                        });
                    });
                }
                @Override
                public void failed(Throwable t, Attachment att2) {
                    handleError(t);
                }
            });
        });
    }
    @Override
    public void failed(Throwable t, Attachment att) {
        handleError(t);
    }
});
// 嵌套层级太深，难以维护和调试！

// Netty 的 ChannelPipeline 解决了这个问题：
// 将复杂的回调逻辑拆分为线性的 Handler 链：
// pipeline.addLast("decoder", new HttpServerCodec())
//          .addLast("handler", new MyBusinessHandler());
```

#### 原因4：Netty 对 NIO 的深度优化已经接近 AIO 的效果

```
Netty 对 NIO 的优化手段：

1. ✅ JVM 层面优化
   - 使用直接内存（Direct Buffer），减少一次内存拷贝
   - 内存池化（ByteBuf pool），避免频繁 GC
   - 对象复用（对象池），减少 GC 压力

2. ✅ OS 层面适配
   - Linux: 自动检测并使用 EpollEventLoop（而非通用 SelectEventLoop）
   - 支持边缘触发 ET 模式（可选开启）
   - TCP 参数自动调优（TCP_NODELAY, SO_KEEPALIVE 等）

3. ✅ 线程模型优化
   - Reactor 主从多线程模型（boss group + worker group）
   - 无锁化串行设计（同一个 Channel 的所有 Handler 在同一线程执行）
   - 任务队列机制平衡负载
```

### 总结：技术选型的权衡

| **因素** | **选择 NIO** | **选择 AIO** |
|----------|-------------|-------------|
| **Linux 支持** | ★★★★★ 成熟完美 | ★★☆☆☆ 不完善 |
| **Windows 支持** | ★★★★☆ 良好 | ★★★★★ IOCP 很强 |
| **性能上限** | ★★★★☆ 足够高 | ★★★★★ 理论最高 |
| **编程复杂度** | ★★★☆☆ 中等 | ★★☆☆☆ 回调地狱 |
| **生态成熟度** | ★★★★★ Netty 生态 | ★★☆☆☆ 少有人用 |
| **调试难度** | ★★★★☆ 可追踪 | ★★☆☆☆ 异步栈难查 |

> **结论**：Netty 选择 NIO 不是因为 AIO 不好，而是因为在当前的技术环境下（尤其是 Linux 服务器为主），**NIO 的综合性价比最高**。如果未来 Linux 的 io_uring 全面普及且 Java 提供良好的支持，Netty 未来可能会转向基于 AIO 的实现。

---

## Q4：什么是 Reactor 模式？什么是 Proactor 模式？两者有什么区别？

**A：**

### Reactor 模式（NIO 采用的模式）

Reactor 模式的核心思想是：**将事件的检测（多路分离）和事件的分发（业务处理）解耦**。

```
┌──────────────────────────────────────────────┐
│                  Reactor 模式                  │
│                                                │
│  ┌─────────┐    事件就绪     ┌─────────────┐ │
│  │  I/O多路  │ ───────────→ │  Reactor     │ │
│  │ 复用器   │   (select/poll)│  (分发器)     │ │
│  │epoll等  │               │              │ │
│  └─────────┘               │  分发到对应    │ │
│                             │  EventHandler │ │
│                             └──────┬───────┘ │
│                                    │          │
│                    ┌───────────────┼────────┐  │
│                    ▼               ▼        ▼  │
│              ┌──────────┐  ┌──────────┐ ┌────┐│
│              │Handler_A │  │Handler_B │ │... ││
│              │处理ACCEPT │  │处理READ  │    ││
│              └──────────┘  └──────────┘ └────┘│
└──────────────────────────────────────────────┘

关键特点：
1. Reactor 通过 select/epoll_wait 监听事件（阻塞在此）
2. 事件就绪后，Reactor 将事件分发给对应的 Handler
3. Handler 负责实际的读写操作（同步地、非阻塞地读写）
4. 应用程序负责 I/O 操作本身
```

### Reactor 的三种变体

```java
// ===== 单 Reactor 单线程（最简单的Reactor）=====
// 所有事情在一个线程中完成：监听事件 + 处理I/O + 业务逻辑
// 优点：简单，无锁
// 缺点：业务逻辑耗时会影响其他连接的处理

// Netty: EventLoopGroup(1) — 类似于这种方式


// ===== 单 Reactor 多线程 =====
// Reactor线程：只负责监听事件 + 分发
// Worker线程池：负责实际的 I/O 操作和业务处理
// 优点：业务逻辑不影响 I/O 监听
// 缺点：Reactor仍是单点（虽然通常不是瓶颈）

// Netty:
// EventLoopGroup bossGroup = new NioEventLoopGroup(1);  // Reactor线程
// EventLoopGroup workerGroup = new NioEventLoopGroup(4); // Worker线程池


// ===== 主从 Reactor 多线程（Netty 默认模式！）=====
// mainReactor: 只负责 ACCEPT 事件（接受新连接）
// subReactor组: 每个subReactor负责一部分连接的 READ/WRITE 事件
// workerThread: 处理耗时业务逻辑（可选）

// Netty 标准配置:
// EventLoopGroup bossGroup = new NioEventLoopGroup(1);   // MainReactor
// EventLoopGroup workerGroup = new NioEventLoopGroup();    // SubReactors
// b.group(bossGroup, workerGroup)

/*
架构图:

           Client Requests
                 │
                 ▼
        ┌─────────────────┐
        │  MainReactor(1)  │  ← 只做 accept，拿到连接后分配给 SubReactor
        │  (Boss Group)    │
        └────────┬────────┘
                 │ 新连接
        ┌────────┴────────┐
        ▼                 ▼
  ┌──────────┐     ┌──────────┐
  │SubReactor│     │SubReactor│   ...
  │  (Worker)│     │  (Worker)│
  └────┬─────┘     └────┬─────┘
       │                │
  READ/WRITE事件    READ/WRITE事件
       │                │
       ▼                ▼
  ┌──────────┐     ┌──────────┐
  │ Handler  │     │ Handler  │  (同一Channel绑定固定SubReactor，无锁)
  └──────────┘     └──────────┘
*/
```

### Proactor 模式（AIO 采用的模式）

```
┌──────────────────────────────────────────────┐
│                  Proactor 模式                 │
│                                                │
│  ┌──────────┐                               │
│  │ Initiator│ 发起异步操作                      │
│  │(应用线程) │ ──async_read/write()────────→   │
│  └──────────┘                               │
│                    ↓                          │
│  ┌──────────────────────────────────────┐     │
│  │         操作系统内核 (Proactor)        │     │
│  │                                      │     │
│  │  1. 接收异步操作请求                   │     │
│  │  2. 执行实际 I/O 操作                  │     │
│  │  3. 将数据拷贝到用户空间               │     │
│  │  4. 完成后通知 CompletionHandler       │     │
│  └───────────────────┬──────────────────┘     │
│                      │ completed callback       │
│                      ▼                          │
│              ┌───────────────┐                  │
│              │CompletionHandler│                │
│              │ (业务处理)      │                │
│              └───────────────┘                  │
│                                                │
│ 关键特点：                                       │
│ 1. 应用程序只需发起异步请求，无需关心 I/O 过程     │
│ 2. 操作系统（Proactor）完成全部 I/O 工作          │
│ 3. 通过回调或信号通知应用"已完成"                 │
│ 4. 应用程序完全不参与 I/O 操作                   │
└──────────────────────────────────────────────┘
```

### Reactor vs Proactor 核心区别

| **维度** | **Reactor (NIO)** | **Proactor (AIO)** |
|----------|-------------------|---------------------|
| **谁做 I/O** | **应用程序自己做**（非阻塞读写） | **操作系统帮着做** |
| **何时知道完成** | 应用主动查询（select/epoll_wait） | 操作系统完成后通知（回调） |
| **角色分工** | Reactor = 事件检测 + 分发 | Proactor = I/O 操作执行者 |
| **关注的是** | "是否可读/可写" | "读/写操作是否完成" |
| **OS 支持** | 所有平台都支持良好 | 仅 Windows IOCP 完善 |
| **Java 对应** | `java.nio.channels.*` | `java.nio.channels.Asynchronous*` |
| **代表框架** | **Netty, Tomcat(NIO), Redis** | 很少有（Windows上的某些服务） |

### 一句话总结

> **Reactor：你问我答（你准备好了吗？好我来读）。**
> **Proactor：你帮我办（你去读，读完了告诉我）。**
>
> **NIO 用 Reactor（同步非阻塞）；AIO 用 Proactor（异步非阻塞）。**

---

## Q5：如何在实际项目中正确选择 BIO/NIO/AIO？

**A：**

### 选型决策树

```
需要网络通信？
│
├─ 连接数少（< 1000）、开发快速优先？
│   └─ YES → BIO（最简单，Tomcat BIO 模式、传统 Socket 编程）
│   └─ NO  ↓
│
├─ 高并发（1000~1000000+）？
│   └─ YES → NIO（首选 Netty 框架）
│   │        ├── HTTP/gRPC/WebSocket → Netty
│   │        ├── RPC 框架 → Dubbo（底层 Netty）
│   │        ├── 消息队列客户端 → RocketMQ/Kafka（底层 Netty）
│   │        └── 游戏服务器 → Netty
│   └─ NO  ↓
│
├─ 大文件异步 I/O？（如日志写入、文件上传下载）
│   └─ YES → AIO (java.nio.file 包下的 Files/API)
│          └─ 示例：Files.asynchronousFileChannel()
│
├─ 需要跨平台且追求极致性能？
│   └─ NO  → NIO（Netty 已足够）
│   └─ YES → 调研当前平台的 AIO 支持情况
│
└─ 最终建议：99%的场景用 NIO (Netty)，AIO 仅用于文件操作
```

### 各场景推荐方案

| **场景** | **推荐方案** | **理由** |
|----------|------------|---------|
| **Web 服务端（Spring Boot 内嵌 Tomcat）** | NIO（Tomcat NIO Connector） | 高并发必备 |
| **RPC 远程调用** | NIO（Netty/Dubbo） | 业界标准 |
| **即时通讯（IM/聊天）** | NIO（Netty + WebSocket） | 长连接+推送 |
| **网关/代理** | NIO（Netty / Zuul / Spring Cloud Gateway） | 高吞吐转发 |
| **大文件上传/下载** | AIO (`AsynchronousFileChannel`) | 文件IO AIO更合适 |
| **日志异步写盘** | AIO 或 NIO MappedByteBuffer | 减少对业务的阻塞 |
| **内部工具/脚本** | BIO（足够简单） | 快速开发 |
| **数据库连接池** | BIO（JDBC 底层是 BIO） | 连接数可控 |

### Netty 推荐配置模板

```java
// 生产环境 Netty 标准配置（NIO Reactor 模式）
public class NettyServerTemplate {

    public static void main(String[] args) throws Exception {
        // === 1. Boss Group（主 Reactor：处理 Accept 事件）===
        // 通常1个线程就够了（纯CPU绑定，不涉及IO等待）
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);

        // === 2. Worker Group（从 Reactor：处理 Read/Write 事件）===
        // 一般设置为 CPU核心数 × 2
        int workerThreads = Runtime.getRuntime().availableProcessors() * 2;
        EventLoopGroup workerGroup = new NioEventLoopGroup(workerThreads);

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();

            bootstrap.group(bossGroup, workerGroup)   // 主从 Reactor
                   .channel(NioServerSocketChannel.class) // NIO Channel
                   .option(ChannelOption.SO_BACKLOG, 1024) // 全连接队列长度
                   .childOption(ChannelOption.TCP_NODELAY, true) // 关闭 Nagle 算法
                   .childOption(ChannelOption.SO_KEEPALIVE, true) // 保持长连接
                   .childOption(ChannelOption.SO_RCVBUF, 64 * 1024) // 接收缓冲区 64KB
                   .childOption(ChannelOption.SO_SNDBUF, 64 * 1024) // 发送缓冲区 64KB
                   .childHandler(new ChannelInitializer<SocketChannel>() {
                       @Override
                       protected void initChannel(SocketChannel ch) {
                           ch.pipeline()
                              .addLast("idleState", new IdleStateHandler(60, 0, 0))
                              .addLast("codec",    new HttpServerCodec())
                              .addLast("aggregator", new HttpObjectAggregator(65536))
                              .addLast("handler",  new BusinessHandler());
                       }
                   });

            ChannelFuture future = bootstrap.bind(8080).sync();
            System.out.println("Netty 服务器启动成功 (NIO Reactor)");
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```
