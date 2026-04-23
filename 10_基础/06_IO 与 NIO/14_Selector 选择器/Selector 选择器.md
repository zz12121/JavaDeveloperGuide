---
title: Selector 选择器
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# Selector 选择器（多路复用）

## 核心作用

Selector 实现 **I/O 多路复用**：一个线程管理**多个 Channel**，监听它们的事件。

```
传统 BIO：每个连接一个线程
Thread-1 ←→ Connection-1
Thread-2 ←→ Connection-2
Thread-3 ←→ Connection-3

NIO + Selector：一个线程管理所有连接
Selector ←→ Connection-1 (监听)
        ←→ Connection-2 (监听)
        ←→ Connection-3 (就绪 → 处理)
```

## 四种事件类型

| 事件 | 常量 | 说明 |
|------|------|------|
| **OP_ACCEPT** | `SelectionKey.OP_ACCEPT` | 服务端接受新连接 |
| **OP_CONNECT** | `SelectionKey.OP_CONNECT` | 客户端连接完成 |
| **OP_READ** | `SelectionKey.OP_READ` | 通道可读 |
| **OP_WRITE** | `SelectionKey.OP_WRITE` | 通道可写 |

## 工作流程

```java
// 1. 创建 Selector
Selector selector = Selector.open();

// 2. 创建 ServerSocketChannel 并配置
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);  // 必须非阻塞

// 3. 注册到 Selector，监听 ACCEPT 事件
server.register(selector, SelectionKey.OP_ACCEPT);

// 4. 循环监听
while (true) {
    // 阻塞直到有事件就绪
    int readyCount = selector.select();

    // 5. 获取就绪事件集合
    Set<SelectionKey> readyKeys = selector.selectedKeys();
    Iterator<SelectionKey> it = readyKeys.iterator();

    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();  // 必须手动移除！

        if (key.isAcceptable()) {
            // 新连接
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // 可读
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int len = client.read(buffer);
            if (len == -1) {
                key.cancel();  // 连接关闭
                client.close();
            } else {
                buffer.flip();
                // 处理数据...
            }
        }
    }
}
```

## SelectionKey

`SelectionKey` 是 Channel 注册到 Selector 后的"令牌"，包含：

| 方法 | 说明 |
|------|------|
| `channel()` | 获取关联的 Channel |
| `selector()` | 获取关联的 Selector |
| `interestOps()` | 获取感兴趣的事件 |
| `readyOps()` | 获取已就绪的事件 |
| `isAcceptable()` | 是否可接受连接 |
| `isReadable()` | 是否可读 |
| `isWritable()` | 是否可写 |
| `attach(obj)` / `attachment()` | 附加对象（如 ByteBuffer） |

## select() 方法

| 方法 | 说明 |
|------|------|
| `select()` | 阻塞直到至少一个事件就绪 |
| `select(long timeout)` | 阻塞最多 timeout 毫秒 |
| `selectNow()` | 不阻塞，立即返回就绪数量 |
| `wakeup()` | 唤醒阻塞的 select() |

## 底层原理

Selector 底层依赖操作系统的 **I/O 多路复用**系统调用：

| 操作系统 | 系统调用 |
|---------|---------|
| Linux | **epoll** |
| macOS | kqueue |
| Windows | select（性能较差） |

## 关联知识点
