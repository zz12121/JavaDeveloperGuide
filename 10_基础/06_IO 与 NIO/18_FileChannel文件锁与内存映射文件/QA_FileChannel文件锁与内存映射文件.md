---
title: FileChannel文件锁与内存映射文件面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# FileChannel文件锁与内存映射文件

## Q1：FileLock 的独占锁和共享锁有什么区别？

**A：**

| 类型 | 效果 | 使用场景 |
|------|------|---------|
| **独占锁（Exclusive）** | 阻止其他进程读写 | 写操作时保护临界资源 |
| **共享锁（Shared）** | 允许其他进程读，阻止写 | 读操作并行化 |

```java
// 独占锁
FileLock exclusiveLock = channel.lock();  // 默认独占

// 共享锁
FileLock sharedLock = channel.lock(0, Long.MAX_VALUE, true);  // 参数 true = shared
```

注意：**FileLock 是进程级别的锁**，同一 JVM 内的多线程会共享同一个锁，因此不适合用于线程间协调（线程间请用 ReentrantLock）。

---

## Q2：tryLock() 和 lock() 的区别？

**A：**

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `lock()` | 阻塞等待，直到获取锁 | 确定性写操作，必须拿到锁 |
| `tryLock()` | 立即返回，获取失败返回 null | 非阻塞场景，防止死锁 |

```java
// lock()：拿不到就等（阻塞）
FileLock lock = channel.lock();
try {
    // 临界区
} finally {
    lock.release();
}

// tryLock()：拿不到就走（非阻塞）
FileLock lock = channel.tryLock();
if (lock != null) {
    try {
        // 临界区
    } finally {
        lock.release();
    }
} else {
    // 锁被占用，跳过或重试
}
```

---

## Q3：内存映射文件（mmap）和普通文件读写的性能差异在哪里？

**A：**

普通 read/write 的数据流向（4次拷贝）：

```
磁盘 → 内核缓冲区(read) → 用户堆内存 → 内核缓冲区(write) → 磁盘
        拷贝1             拷贝2           拷贝3            拷贝4
```

mmap 的数据流向（2次拷贝）：

```
磁盘 → 内核缓冲区 → 用户直接内存（虚拟地址映射同一块物理内存）→ 磁盘
        拷贝1                      （无需额外拷贝）
```

**性能差异来源**：
1. **减少内存拷贝次数**：4次 → 2次
2. **减少上下文切换**：mmap 读写在用户态完成，不需要每次都进入内核态
3. **按需加载**：OS 的 page cache 自动管理，只加载需要的页面

**实测性能**：大文件随机读写场景下，mmap 比普通读写快 **30%~50%**。

---

## Q4：MappedByteBuffer 的 force() 方法有什么作用？

**A：**

`force()` 将内存中的修改**强制刷回磁盘**，确保数据持久化。

```java
MappedByteBuffer buffer = channel.map(..., 0, size);
buffer.put(0, (byte)'X');  // 修改内存

buffer.force();  // 立即刷盘（同步，等待磁盘写入完成）

// JDK 11+
buffer.force(true);  // 参数 true = 包含元数据
```

**为什么不自动刷盘？**
- mmap 的修改由 OS 的 page cache 管理，OS 会在合适的时机批量写回
- 如果 JVM 崩溃，OS 通常能保证已刷盘的数据（但不是100%可靠）
- **关键数据写入后必须 force()**，防止 OS 未及时刷盘导致数据丢失

**性能注意**：`force()` 是同步的，频繁调用会影响性能。适合在事务边界或关键节点调用。

---

## Q5：mmap 有什么局限性？什么时候适合用 mmap？

**A：**

**局限性**：
1. **文件大小有限制**：最大映射大小受 OS 进程虚拟地址空间限制（Linux 通常约 128TB）
2. **小文件反而更慢**：mmap 有映射开销，适合大文件（>1MB）
3. **页故障风险**：访问未加载的映射区域会触发 page fault（磁盘 I/O），影响性能
4. **内存泄漏风险**：MappedByteBuffer 在 GC 前不会释放映射，生命周期管理复杂
5. **跨平台差异**：Windows 上 mmap 性能不如 Linux

**适合场景**：
- 大文件随机读写（日志分析、数据库存储引擎、缓存索引）
- 需要高性能持久化的大数据结构（如 Lucene 的 FDB）
- 多进程共享同一份文件数据

**不适合场景**：
- 小文件（<1MB）
- 顺序写入为主的场景
- 频繁创建销毁的临时文件

---

## Q6：FileChannel 的 transferTo() 为什么快？

**A：**

`transferTo()` 利用 OS 的 **零拷贝（Zero-Copy）** 技术——sendfile 系统调用：

```
传统方式：
磁盘 → 内核缓冲区 → 用户堆内存 → Socket缓冲区 → 网卡（4次拷贝）

transferTo（零拷贝）：
磁盘 → 内核缓冲区 → 网卡（2次拷贝）
```

```java
// 传统：逐字节读入用户内存
FileChannel in = new FileInputStream("bigfile").getChannel();
FileChannel out = new FileOutputStream("dest").getChannel();
ByteBuffer buf = ByteBuffer.allocate(8192);
while (in.read(buf) != -1) {
    buf.flip();
    out.write(buf);
    buf.clear();
}

// 零拷贝：直接在内核态传输
long bytesTransferred = in.transferTo(0, in.size(), out);
// 自动利用 sendfile，最大单次 2GB
```

**sendfile 的限制**：
- Linux 单次最多传输 2GB，超出需要循环调用
- 中间没有经过用户态，所以无法在传输过程中修改数据

---

## Q7：NIO 的 Scatter/Gather 是什么？有什么用？

**A：**

- **Scatter（分散读取）**：一个 `read()` 将数据填充到**多个 Buffer**
- **Gather（聚集写入）**：一个 `write()` 将**多个 Buffer** 的数据写入通道

```java
// Scatter — 适合解析固定格式协议（头部 + 正文）
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);
channel.read(new ByteBuffer[]{ header, body });

// Gather — 适合拼包发送
ByteBuffer prefix = ByteBuffer.wrap("HTTP/1.1 200 OK\r\n".getBytes());
ByteBuffer content = ByteBuffer.wrap("Hello".getBytes());
channel.write(new ByteBuffer[]{ prefix, content });
```

**优势**：
- 避免了手动拼包/拆包的操作
- 减少内存拷贝次数
- 在网络协议解析（如 HTTP、WebSocket）中非常有用
