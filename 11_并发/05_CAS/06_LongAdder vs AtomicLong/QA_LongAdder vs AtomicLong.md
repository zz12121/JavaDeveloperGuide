---
title: LongAdder vs AtomicLong
tags:
  - Java/并发
  - 问答
  - 对比型
module: 05_CAS
created: 2026-04-18
---

# LongAdder vs AtomicLong（LongAdder分段CAS解决高并发热点，适合统计场景）

## Q1：LongAdder 和 AtomicLong 有什么区别？

**A**：核心区别在于并发策略：

| 维度 | AtomicLong | LongAdder |
|------|-----------|-----------|
| 竞争策略 | 所有线程 CAS 同一个变量 | 分散到多个 Cell，每个 Cell 独立 CAS |
| 高并发性能 | CAS 冲突严重，CPU 空转 | 分段减少冲突，性能好 5~10 倍 |
| 实时精确值 | ✅ 支持 | ⚠️ sum() 不保证精确 |
| compareAndSet | ✅ 支持 | ❌ 不支持 |
| 适用场景 | 精确控制（状态标记、序号） | 统计累加（QPS、计数器） |

---

## Q2：LongAdder 是怎么解决热点竞争的？

**A**：通过**分段 CAS**（Cell 数组）：

1. **无竞争**：直接 CAS 更新 `base` 字段（和 AtomicLong 一样）
2. **出现竞争**：创建 cells 数组，每个线程通过 `probe & (length-1)` 哈希到自己的 Cell
3. **Cell 内 CAS**：线程只 CAS 自己的 Cell，大幅减少冲突
4. **求和**：`sum()` 遍历 base + 所有 Cell 的值求和

每个 Cell 用 `@Contended` 注解避免伪共享（缓存行填充）。

---

## Q3：什么场景用 AtomicLong，什么场景用 LongAdder？

**A**：

**用 AtomicLong**：
- 需要 `compareAndSet` 操作（如状态机控制）
- 需要精确的实时值（如序列号生成器）
- 并发度低（< 10 个线程）

**用 LongAdder**：
- 只需要累加统计（如接口调用量、QPS、在线人数）
- 高并发写入但低频读取
- 允许 sum() 有微小误差

```java
// AtomicLong：序号生成
AtomicLong sequence = new AtomicLong(0);
long nextId = sequence.incrementAndGet(); // 需要精确值

// LongAdder：QPS 统计
LongAdder requestCount = new LongAdder();
requestCount.increment(); // 每次请求 +1
long qps = requestCount.sumThenReset(); // 每秒统计一次
```

---

## Q4：什么是伪共享？为什么 LongAdder.Cell 要用 @Contended？

**A**：伪共享是指两个无关变量恰好在同一个缓存行（64 bytes），一个被频繁 CAS 会导致另一个的缓存行失效：

```java
// ❌ 伪共享：x 和 y 在同一缓存行
class NoPadding {
    volatile long x;
    volatile long y;
}
// 线程1 CAS x → y 的缓存行也失效
// 线程2 读 y → 缓存 miss，必须重新加载

// ✅ 独占缓存行
class WithPadding {
    @jdk.internal.vm.annotation.Contended
    volatile long x;
    @jdk.internal.vm.annotation.Contended
    volatile long y;
}
// 编译器自动在 x 和 y 之间填充 128 bytes
// 线程1 CAS x → y 完全不受影响
```

**LongAdder 使用 @Contended 的原因**：100 个线程 CAS 100 个不同的 Cell，如果它们挤在同一缓存行，一个 Cell 被修改会导致其他 Cell 所在行失效，引发"伪共享风暴"，性能骤降。

---

## Q5：@Contended 注解的填充规则是什么？有什么限制？

**A**：

```java
// @Contended 填充规则（JDK8+）
// - 默认在字段前后各填充 64 bytes（总计 128 bytes）
// - 需要手动启用：-XX:-RestrictContended（默认开启）
// - JDK8 可调整填充宽度：-XX:ContendedPaddingWidth=64
// - JDK9+ 固定为 128 bytes，不可调整

// 使用注意
// 1. 仅对 HotSpot JVM 有效
// 2. 仅对加了 @Contended 的字段本身生效（父类字段不会）
// 3. 类加载时 HOTSPOT 会自动插入填充代码（不是运行时）

// 验证缓存行大小
System.out.println(sun.misc.Unsafe.ARRAY_BYTE_INDEX_SCALE); // 1
// 或通过：getconf LEVEL1_DCACHE_LINESIZE（Linux）
```

---

## Q6：LongAdder 的 cells 数组为什么最大长度是 CPU 核数？

**A**：这是 Striped64 的设计哲学：

```java
// cells 最大长度 = NCPU（CPU 核数）
static final int NCPU = Runtime.getRuntime().availableProcessors();

// 为什么不超过 NCPU？
// 1. 超过 CPU 核数的 Cell 没有意义——同一时刻最多 NCPU 个线程并行
// 2. 减少内存开销：cells 是 CPU 敏感数据结构
// 3. 减少伪共享风险：Cell 数量越少，落在同一缓存行的概率越高

// cells 数组长度 = 2 的幂次，便于 hash 取模
// probe = ThreadLocalRandom.current().getProbe();
// index = (cells.length - 1) & probe;  // 替代 % 取模
```

## 关联知识点

