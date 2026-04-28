
# 直接内存（Direct Memory）

## Q1：什么是直接内存？

**A**：直接内存（Direct Memory）是 JVM 堆外的本地内存，通过 `ByteBuffer.allocateDirect()` 分配。不受 JVM 堆大小限制，但受物理内存和 `-XX:MaxDirectMemorySize` 限制。

---

## Q2：直接内存和堆内存的区别？

**A**：

| 区别 | 直接内存 | 堆内存 |
|------|----------|--------|
| **位置** | 本地内存（堆外） | JVM 堆 |
| **GC** | 不受 GC 管理 | 受 GC 管理 |
| **分配速度** | 较慢 | 较快 |
| **IO 性能** | 高（零拷贝） | 低（需拷贝） |
| **释放方式** | 手动/GC 时调用 Cleaner | 自动 |

---

## Q3：为什么 NIO 使用直接内存？

**A**：NIO 使用直接内存实现**零拷贝**：
- **传统 IO**：磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡（2次拷贝）
- **NIO 直接内存**：磁盘 → 内核缓冲区 → 网卡（1次拷贝，直接从内核到网卡）

**优势**：
1. 减少数据在堆内外的复制
2. 提高 IO 操作性能
3. 适合高并发、大数据量场景（如 Netty）

---

## Q4：直接内存会导致 OOM 吗？

**A**：会。直接内存 OOM：
- `-XX:MaxDirectMemorySize` 默认等于 `-Xmx`
- 如果分配过多直接内存，物理内存耗尽会 OOM
- 常见于 Netty 等高性能框架

```java
// 不断分配直接内存
while (true) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);
}
```

---

## Q5：如何查看直接内存使用？

**A**：
```bash
# JMX 监控
jconsole

# Arthas
dashboard

# 命令行
java -XX:MaxDirectMemorySize=512m YourApp
```

---

## Q6：如何正确释放直接内存？

**A**：直接内存通过 `sun.misc.Cleaner` 释放：
- **方式1**：依赖 GC，GC 时 Cleaner 检测并释放
- **方式2**：手动调用 `buffer = null` 加速回收
- **方式3**：Netty 等框架使用池化技术复用

```java
// 主动释放
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
// 方式1
buffer = null; // 等待 GC 回收
// 方式2（反射）
sun.misc.Cleaner.clean(buffer);
```

---

## Q7：设置 `-Xmx` 能限制直接内存吗？

**A**：不能。`-Xmx` 只限制堆大小，直接内存不受其限制。必须单独设置 `-XX:MaxDirectMemorySize`。

---
