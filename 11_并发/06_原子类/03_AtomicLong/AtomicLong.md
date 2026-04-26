---
title: AtomicLong
tags:
  - Java/并发
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicLong（64位原子操作，解决Long型读写的线程安全问题）

## 先说结论

`AtomicLong` 提供对 64 位 long 类型的原子操作，底层通过 `Unsafe.compareAndSetLong` 实现。在 32 位 JVM 上，long 的读写本身不是原子的（分两次 32 位操作），`AtomicLong` 的 volatile + CAS 确保了跨平台的原子性。JDK8+ 高并发统计场景推荐使用 `LongAdder` 替代。

## 深度解析

### 为什么需要 AtomicLong

在 32 位 JVM 上，一个 long 变量占 64 位（8 字节），一次只能读写 32 位：

```
线程1: 写入 long = 0x00000000_FFFFFFFF
  第一次写高32位: 0x00000000_xxxxxxxx
  → 线程2 读取 → 可能读到 0x00000000_00000000（只读了高32位）

线程1: 第二次写低32位: 0x00000000_FFFFFFFF
```

`AtomicLong` 通过 CAS 指令保证读-改-写的原子性。

### 核心方法

```java
public class AtomicLong {
    private volatile long value;

    // 与 AtomicInteger 相同的方法签名，类型改为 long
    public final long get();
    public final void set(long newValue);
    public final boolean compareAndSet(long expected, long newValue);
    public final long getAndIncrement();
    public final long incrementAndGet();
    public final long getAndAdd(long delta);
    public final long addAndGet(long delta);
    public final long getAndSet(long newValue);

    // JDK8+ 函数式操作
    public final long updateAndGet(LongUnaryOperator updateFunction);
    public final long accumulateAndGet(long x, LongBinaryOperator accumulatorFunction);
}
```

### AtomicLong vs Long vs volatile long

| 维度 | long | volatile long | AtomicLong |
|------|------|--------------|-----------|
| 原子写入（32位JVM） | ❌ | ❌ | ✅ CAS |
| 原子读-改-写 | ❌ | ❌ | ✅ CAS |
| 可见性 | ❌ | ✅ | ✅ volatile |
| 复合操作 | ❌ | ❌ | ✅ CAS |

## 易错点/踩坑

- ❌ 64 位 JVM 上 AtomicLong 就多余——即使 64 位 JVM，`i++` 仍不是原子的
- ❌ AtomicLong 在高并发下性能一定好——竞争激烈时 CAS 自旋浪费 CPU
- ✅ 高并发统计场景优先用 `LongAdder`

## 代码示例

```java
// 序列号生成器
public class SequenceGenerator {
    private final AtomicLong sequence = new AtomicLong(0);

    public long nextId() {
        return sequence.incrementAndGet();
    }

    // 带过期重置的计数器
    public long getAndResetIfExpired(long expireTime) {
        long current;
        do {
            current = sequence.get();
            if (System.currentTimeMillis() > expireTime) {
                if (sequence.compareAndSet(current, 0)) {
                    return 0;
                }
            }
        } while (true);
    }
}
```

## 关联知识点
