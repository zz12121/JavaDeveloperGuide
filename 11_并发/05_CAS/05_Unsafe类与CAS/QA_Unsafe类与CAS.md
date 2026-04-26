---
title: Unsafe类与CAS
tags:
  - Java/并发
  - 问答
  - 源码型
module: 05_CAS
created: 2026-04-18
---

# Unsafe类与CAS（sun.misc.Unsafe，底层CAS实现，直接操作内存）

## Q1：Unsafe 类是什么？有哪些核心能力？

**A**：`Unsafe` 是 JDK 内部提供的底层操作工具类，核心能力包括：

1. **CAS 操作**：`compareAndSetInt/Long/Object`，原子类的基础
2. **直接内存操作**：`allocateMemory/freeMemory`，DirectByteBuffer 的底层
3. **字段偏移量**：`objectFieldOffset`，通过偏移量直接操作字段
4. **线程调度**：`park/unpark`，LockSupport 的底层
5. **内存屏障**：`loadFence/storeFence/fullFence`
6. **类操作**：`allocateInstance`（不调用构造器创建对象）

JDK9 后从 `sun.misc.Unsafe` 迁移到 `jdk.internal.misc.Unsafe`。

---

## Q2：Unsafe 的 CAS 方法如何工作？为什么要字段偏移量？

**A**：CAS 通过**对象引用 + 字段偏移量**定位到内存中的具体字段：

```java
// 获取偏移量
long offset = U.objectFieldOffset(MyClass.class, "count");

// 通过偏移量直接操作字段内存
U.compareAndSetInt(obj, offset, 10, 20);
```

偏移量 = 字段在对象内存中的字节偏移位置。这样 Unsafe 可以绕过 Java 访问控制，直接对任意内存地址执行硬件级 CAS，效率高于反射。

---

## Q3：为什么业务代码不建议直接使用 Unsafe？

**A**：

1. **不安全**：直接操作内存，越界访问会导致 JVM 崩溃
2. **API 不稳定**：不同 JDK 版本可能变更，`sun.misc` 包不属于 Java SE 规范
3. **可读性差**：偏移量操作不直观，维护成本高
4. **有替代方案**：`AtomicInteger`、`VarHandle`（JDK9+）封装了 Unsafe 功能，更安全

```java
// JDK9+ 推荐使用 VarHandle 替代 Unsafe
VarHandle VH = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "count", int.class);
VH.compareAndSet(obj, 10, 20); // 比 Unsafe 更安全
```

---

## Q4：VarHandle 和 Unsafe、AtomicInteger 的区别？什么时候用哪个？

**A**：

| 维度 | Unsafe | VarHandle | AtomicInteger |
|------|--------|-----------|---------------|
| API 稳定性 | ❌ 不稳定（内部类） | ✅ 标准 API（jdk.incubator.foreign → jdk.lang.invoke） | ✅ 稳定 |
| JDK 版本 | 所有版本 | JDK9+ | 所有版本 |
| 安全性 | ❌ 可越界访问 | ✅ 有边界检查 | ✅ 最安全 |
| 功能 | 最强（内存/CAS/调度） | 强（多种原子/屏障操作） | 专用（int/long） |
| 学习成本 | 高 | 中 | 低 |

```java
// 业务代码首选 AtomicInteger
AtomicInteger counter = new AtomicInteger(0);

// 需要自定义原子操作时用 VarHandle（JDK9+）
VarHandle VH = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "value", long.class);
VH.compareAndSet(this, 0L, 1L);

//Unsafe 仅限 JDK 内部使用（框架层）
```

---

## Q5：Unsafe 在实际框架中有哪些经典应用？

**A**：

| 框架 | Unsafe 使用方式 | 作用 |
|------|----------------|------|
| **Netty** | `allocateMemory()` 堆外内存 | DirectByteBuffer，零拷贝，绕过 GC |
| **Kryo/Fastjson** | `allocateInstance()` | 不调用构造器，快速序列化 |
| **Disruptor** | CAS + 自旋 | 无锁队列核心机制 |
| **Hadoop** | 堆外 I/O 缓冲 | 减少 GC 压力 |
| **JDK 内部** | 原子类、AQS、ConcurrentHashMap | 所有 JUC 底层 |

```java
// Netty 的 DirectByteBuffer 原理
// ByteBuffer.allocateDirect() 底层调用:
// unsafe.allocateMemory(size) → 分配堆外内存
// unsafe.setMemory(address, size, (byte) 0) → 初始化

// Kryo 的对象创建（跳过构造器）
Object obj = unsafe.allocateInstance(MyClass.class);
// 构造器中的初始化逻辑被跳过，字段为默认值
// 需要手动调用反射或特殊方法初始化字段
```

