---
title: FileChannel文件锁与内存映射文件
tags:
  - Java/IO
  - 原理型
  - NIO
module: 06_IO与NIO
created: 2026-04-25
---

# FileChannel文件锁与内存映射文件

## FileChannel 基础

### FileChannel 的特点

- FileChannel 是同步的（相对于 AIO 的 AsynchronousFileChannel）
- 不能设置为非阻塞（只能阻塞模式）
- 线程安全（多个线程可以并发读同一个 FileChannel）
- 支持文件锁和内存映射

### 获取 FileChannel

```java
// 方式1：从 FileInputStream/FileOutputStream
FileInputStream fis = new FileInputStream("file.txt");
FileOutputStream fos = new FileOutputStream("file.txt");
FileChannel fcIn = fis.getChannel();
FileChannel fcOut = fos.getChannel();

// 方式2：从 RandomAccessFile
RandomAccessFile raf = new RandomAccessFile("file.txt", "rw");
FileChannel channel = raf.getChannel();

// 方式3：FileChannel.open()（JDK 7+，推荐）
FileChannel channel = FileChannel.open(
    Paths.get("file.txt"),
    StandardOpenOption.READ,
    StandardOpenOption.WRITE,
    StandardOpenOption.CREATE
);
```

## 文件锁（FileLock）

### 什么是文件锁

- 文件锁用于**多进程或多线程**间的文件访问协调
- 锁定的是**文件区域**，不是文件本身
- JVM 进程退出后，文件锁自动释放

### 锁的类型

| 类型 | 说明 | 对应方法 |
|------|------|:---------:|
| 独占锁（Exclusive Lock） | 阻止其他进程读写 | `lock()` |
| 共享锁（Shared Lock） | 允许其他进程共享读，但阻止写 | `lock(long, long, true)` |

### 基本使用

```java
FileChannel channel = new RandomAccessFile("data.txt", "rw").getChannel();

// 获取独占锁（阻止其他进程读写）
FileLock lock = channel.lock();
try {
    // 临界区操作
    channel.write(ByteBuffer.wrap("data".getBytes()));
} finally {
    lock.release();  // 释放锁
}

// 获取共享锁（允许其他进程读）
FileLock sharedLock = channel.lock(0, Long.MAX_VALUE, true);
```

### tryLock()（非阻塞）

```java
// 尝试获取锁，立即返回（不阻塞）
FileLock lock = channel.tryLock();
if (lock != null) {
    // 获取成功
    try {
        // ...
    } finally {
        lock.release();
    }
} else {
    // 获取失败（已被其他进程持有）
}
```

### 文件锁的注意事项

1. **不同 JVM 进程**之间的锁才有效，同一 JVM 的多线程共享锁
2. Windows 上独占锁是强制性的，共享锁则依赖 OS
3. 锁的释放要放在 `finally` 中确保执行
4. **不要在持有锁期间执行阻塞 I/O**，否则影响其他等待锁的线程

### 锁区域

```java
// 锁定文件的一部分
FileLock lock = channel.lock(start, size, shared);
// start: 锁定起始位置
// size:  锁定字节数（0 表示到文件末尾）
// shared: true=共享锁, false=独占锁
```

## 内存映射文件（MappedByteBuffer）

### 什么是内存映射

- 通过 `FileChannel.map()` 将文件直接映射到 JVM 堆外内存
- 读写文件像操作内存数组一样，性能接近原生 I/O
- 本质是 OS 的 mmap 系统调用

### 基本使用

```java
FileChannel channel = new RandomAccessFile("data.txt", "rw").getChannel();

// 将文件映射到内存（只读模式）
MappedByteBuffer roBuffer = channel.map(
    FileChannel.MapMode.READ_ONLY,  // 只读
    0,                               // 起始位置
    channel.size()                   // 映射大小
);

// 读写模式
MappedByteBuffer rwBuffer = channel.map(
    FileChannel.MapMode.READ_WRITE, // 可读写
    0,
    channel.size()
);

// 写入（直接修改内存，数据会在合适时机写回磁盘）
rwBuffer.put(0, (byte)'X');

// 强制刷新到磁盘（同步）
rwBuffer.force();
```

### 内存映射 vs 普通 FileChannel 读写

| 维度 | 内存映射（mmap） | 普通读写（read/write） |
|------|:---:|:---:|
| 拷贝次数 | 1次（磁盘→内核→直接内存） | 3-4次 |
| 读取方式 | 像数组一样直接访问 | 系统调用 read() |
| 大文件处理 | 非常适合 | 需多次系统调用 |
| 小文件 | 额外开销大，不适合 | 适合 |
| GC 影响 | 堆外内存，不受 GC 直接管理 | 堆内存 |
| 写入保证 | 需手动 force() 刷盘 | close() 时自动刷盘 |
| 适用场景 | 大文件随机读写（数据库、日志） | 常规文件操作 |

### force() 强制刷盘

```java
MappedByteBuffer buffer = channel.map(..., 0, size);
buffer.put(data);

// 只刷新文件元数据（修改时间等）
buffer.force();
```

## FileChannel 读写操作

### 分散读取 / 聚集写入

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body = ByteBuffer.allocate(1024);

// 分散读取（Scatter）
ByteBuffer[] buffers = { header, body };
channel.read(buffers);  // 数据依次填满 header 再填 body

// 聚集写入（Gather）
header.flip();
body.flip();
channel.write(new ByteBuffer[]{ header, body });
```

### position() 和 size()

```java
// 获取当前 position
long pos = channel.position();

// 设置 position
channel.position(pos + 10);

// 获取文件大小
long size = channel.size();

// 截断文件（删除 position 之后的内容）
channel.truncate(100);
```

## 关联知识点

