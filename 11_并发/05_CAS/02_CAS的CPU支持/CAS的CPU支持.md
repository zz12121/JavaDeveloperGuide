---
title: CAS的CPU支持
tags:
  - Java/并发
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS的CPU支持（X86的CMPXCHG指令，原子完成比较和交换）

## 先说结论

CAS 的底层依赖 CPU 提供的原子指令：X86 架构使用 `CMPXCHG` 指令，ARM 架构使用 `LDXR/STXR`（LL/SC）指令。JVM 通过 `sun.misc.Unsafe` 类的 native 方法调用这些 CPU 指令，从而在 Java 层面实现硬件级别的原子操作。

## 深度解析

### X86：CMPXCHG 指令

```
CMPXCHG dest, src
; 伪代码：
; if (EAX == dest) {
;     ZF = 1
;     dest = src
; } else {
;     ZF = 0
;     EAX = dest
; }
```

- `EAX` 寄存器存放期望值（Expected）
- `dest` 是内存位置（V）
- `src` 是新值（New）
- 结果通过 ZF 标志位判断成功/失败

### 锁总线 vs 缓存锁

CPU 执行 CMPXCHG 时通过两种方式保证多核原子性：

```
方式一：锁总线（旧）
┌─────────────────────────────────────────┐
│  LOCK CMPXCHG [mem], reg                │
│  → LOCK 信号拉低总线                    │
│  → 其他 CPU 无法访问内存                │
│  → 粒度粗，性能差                       │
└─────────────────────────────────────────┘

方式二：缓存锁（新，P6+）
┌─────────────────────────────────────────┐
│  CMPXCHG [mem], reg                    │
│  → 自动使用 MESI 协议锁定缓存行         │
│  → 只阻塞同一缓存行的其他 CPU           │
│  → 粒度细，性能好                       │
└─────────────────────────────────────────┘
```

### ARM：LL/SC 指令对

```
LDXR W0, [X1]     ; Load Exclusive：加载值到 W0，并标记此缓存行
ADD  W0, W0, #1   ; 计算新值
STXR W2, W0, [X1] ; Store Exclusive：尝试存储，成功则 W2=0
CBNZ W2, retry    ; 失败则重试
```

### JVM 层的封装

```
Java 层                    Native 层                  CPU 指令
Unsafe.compareAndSwapInt → unsafe.cpp::cmpxchg → CMPXCHG
AtomicInteger.getAndAdd   → unsafe.cpp::getAndAddInt → XADD
```

## 易错点/踩坑

- ❌ 认为 Java 层面用 synchronized 实现了 CAS——CAS 是硬件指令，不需要 Java 锁
- ❌ 认为 CMPXCHG 没有性能开销——缓存锁仍有 MESI 协议的通信成本
- ✅ CAS 的性能瓶颈在于缓存一致性协议的开销，而非指令本身

## 代码示例

```java
// Unsafe 中的 CAS 方法签名（JDK17+）
// JDK9+ 改为 jdk.internal.misc.Unsafe
public final native boolean compareAndSetInt(
    Object o,       // 对象
    long offset,    // 字段偏移量
    int expected,   // 期望值
    int newValue    // 新值
);

// AtomicInteger 底层调用
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");

private volatile int value;

public final boolean compareAndSet(int expected, int newValue) {
    return U.compareAndSetInt(this, VALUE, expected, newValue);
}
```

### MESI 缓存一致性协议

MESI（Modified/Exclusive/Shared/Invalid）是 CPU 多核缓存一致性协议，确保同一缓存行数据在多核间的一致性：

```
缓存行四种状态：

M (Modified) — 已修改
  └─ 本核独占且已脏，数据与主存不一致
  
E (Exclusive) — 独占
  └─ 本核独占，数据与主存一致
  
S (Shared) — 共享
  └─ 多核共享，数据与主存一致
  
I (Invalid) — 无效
  └─ 缓存行已被其他核修改，本核副本失效
```

**状态转换流程**：

```
CPU0 读变量 X（缓存 miss）
  → 从主存加载 X
  → 状态：E（独占）

CPU1 读变量 X（缓存 miss）
  → CPU0 响应，数据返回
  → CPU0 & CPU1 状态：S（共享）

CPU0 写入 X
  → CPU0 通知 CPU1 使其缓存行失效
  → CPU0 状态：M（已修改）
  → CPU1 状态：I（无效）

CPU0 写回主存
  → 状态：E（独占，数据与主存一致）
```

**CAS 与 MESI 的关系**：

```
CMPXCHG [x], eax 的执行过程（MESI 视角）：
1. 总线嗅探：CPU 检测 x 是否在其他核的缓存中
2. 如果在其他核（状态 S）→ 触发 RFO（Request For Ownership）
   → 其他核将缓存行置为 I（无效）
   → 本核获得缓存行所有权（状态变为 E 或 M）
3. 执行比较和写入
4. 如果本核已独占（M）→ 直接修改，无需总线通信
```

**MESI 协议的瓶颈**：

- 每次 CAS 需要总线嗅探，高并发下总线带宽成为瓶颈
- 伪共享：两个无关变量恰好在同一个缓存行，一个被频繁 CAS，另一个被频繁读取
- 解决方案：`@Contended` 注解填充缓存行

### 伪共享与缓存行填充

**什么是伪共享**：

```
假如同一个缓存行（64 bytes）存放了两个变量：
┌──────────────────────────────────────────┐
│ x (8 bytes)    y (8 bytes)   填充...    │ ← 同一缓存行
└──────────────────────────────────────────┘
  ↑
  线程1 CAS(x)   线程2 CAS(y)
  → x 被修改，整行失效（包括 y！）
  → 线程2 的 y 缓存行失效，必须重新从主存加载
  → 性能骤降
```

**解决：缓存行填充**：

```java
// LongAdder 的 Cell 使用 @Contended 独占缓存行
@sun.internal.annotation.Contended
static final class Cell {
    volatile long value;
    // 编译时自动填充 ~128 bytes，确保不与其他变量共享缓存行
}

// 手动填充示例
class PaddingObject {
    long p1, p2, p3, p4, p5, p6, p7, p8; // 填充 64 bytes
    volatile long value;
    long q1, q2, q3, q4, q5, q6, q7, q8; // 填充 64 bytes
}
```

**@Contended 注解原理**：

```java
// JDK8+ 提供 @Contended
// 默认填充 128 bytes（可通过 -XX:ContendedPaddingWidth 调整）
// 使用场景：LongAdder.Cell、ThreadLocalRandom 的 ThreadLocalMap.Entry

// 验证缓存行大小
// Linux:  getconf LEVEL1_DCACHE_LINESIZE → 64
// Windows: wmic cpu get L3CacheLineSize

// 性能影响
// 无填充：100线程 CAS → 100万次失败/秒
// 有填充：100线程 CAS → 几乎无失败
```

## 关联知识点

