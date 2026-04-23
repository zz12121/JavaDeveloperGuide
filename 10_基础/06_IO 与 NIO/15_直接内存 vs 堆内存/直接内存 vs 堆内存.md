---
title: 直接内存 vs 堆内存
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# 直接内存 vs 堆内存

## 两种内存

NIO 的 `ByteBuffer` 支持两种分配方式：

```java
// 堆内存（Heap Buffer）
ByteBuffer heapBuffer = ByteBuffer.allocate(1024);

// 直接内存（Direct Buffer）
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);
```

## 核心对比

| 维度 | 堆内存 Buffer | 直接内存 Buffer |
|------|-------------|----------------|
| 分配位置 | JVM **堆** | **操作系统内存**（堆外） |
| 分配/释放 | 快（JVM 管理） | 慢（OS 系统调用） |
| 读写速度 | 普通 | **更快**（少一次拷贝） |
| GC 影响 | 受 GC 管理 | 不受 GC 管理 |
| 内存占用 | 受限于 JVM 堆大小 | 不受堆大小限制 |
| 创建开销 | 小 | **大**（分配+初始化慢） |
| 获取方式 | `allocate()` | `allocateDirect()` |

## IO 过程对比

### 堆内存 Buffer 的 I/O

```
磁盘 → OS 内核缓冲区 → JVM 堆内存（Buffer）→ JVM 用户代码
        ↑ 一次拷贝              ↑ 一次拷贝
        （DMA）                （CPU）
```

### 直接内存 Buffer 的 I/O

```
磁盘 → OS 内核缓冲区 → 直接内存（Buffer）→ JVM 用户代码
        ↑ 一次拷贝（DMA）  ↑
        无额外拷贝！直接内存可以被 OS 直接访问
```

直接内存省去了**内核缓冲区 → JVM 堆**的拷贝，因此 I/O 性能更好。

## 直接内存的适用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| 大文件读写 | ✅ 直接内存 | 减少 I/O 拷贝 |
| 网络通信（高并发） | ✅ 直接内存 | 减少内核→JVM 拷贝 |
| 生命周期的短期 Buffer | ❌ 堆内存 | 直接内存分配开销大 |
| 频繁创建/销毁 | ❌ 堆内存 | 直接内存释放依赖 GC |

## 直接内存的风险

1. **内存泄漏**：直接内存不受 JVM 堆管理，如果 `DirectByteBuffer` 没有被 GC 回收，对应的堆外内存不会释放
2. **OutOfMemoryError**：直接内存占用过多时也会 OOM（但不影响堆大小设置）
3. **排查困难**：堆外内存问题不像堆内存那样容易通过 jmap 等工具排查

## 调整直接内存大小

```bash
# 限制直接内存最大值（默认与 -Xmx 相等）
-XX:MaxDirectMemorySize=256M
```

## 关联知识点