---
title: 伪共享
tags:
  - Java/并发
  - 问答
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# 伪共享（缓存行竞争导致性能问题）

## Q1：什么是伪共享？它和真正的共享有什么区别？

**A**：伪共享（False Sharing）是指多个线程修改**逻辑上独立**但**物理上位于同一缓存行**的变量，导致 CPU 缓存一致性协议（如 MESI）反复使缓存行失效，引发大量 Cache Miss，性能严重下降。

与真正的数据共享（多个线程读写同一个变量）不同，伪共享中的变量在逻辑上互不相关，问题出在 CPU 缓存行粒度（通常 64 字节）远大于单个变量的粒度。例如两个 `volatile long` 变量仅占 16 字节，必然落入同一个缓存行，即使线程各改各的，也会互相拖慢。

---

## Q2：如何检测伪共享问题？

**A**：常见检测方法：

1. **性能分析工具**：使用 Linux `perf stat -e cache-misses` 统计缓存未命中次数，伪共享场景下 Cache Miss 会异常偏高。
2. **JMH 基准测试**：对可疑代码用 JMH 做微基准测试，对比单线程和多线程吞吐量差异。
3. **代码审查**：关注多线程频繁更新的紧凑数据结构（如数组相邻元素、同一个类中的多个 volatile 字段）。

```bash
# Linux 下使用 perf 检测缓存事件
perf stat -e cache-misses,L1-dcache-loads,L1-dcache-load-misses java YourApp
```

---

## Q3：有哪些常见的解决方案？

**A**：三种主要方案：

**方案一：手动缓存行填充（Padding）**

在变量之间插入无用字段，使竞争变量位于不同缓存行：

```java
public class PaddedCounter {
    public volatile long value;
    // 填充 7 个 long（56 字节）+ 对象头 12 字节，合计超过 64 字节
    public long p1, p2, p3, p4, p5, p6, p7;
}
```

**方案二：@Contended 注解（JDK 8+）**

JVM 自动在字段/类周围插入填充：

```java
public class ContendedDemo {
    @jdk.internal.vm.annotation.Contended
    public volatile long value;
}
// 启动参数：-XX:-RestrictContended
```

**方案三：使用 LongAdder 替代 AtomicLong**

`LongAdder` 内部 Cell 数组通过 `@Contended` 注解分散竞争，天然避免伪共享：

```java
// 推荐：高并发计数场景
LongAdder counter = new LongAdder();
counter.increment(); // 内部按线程分散到不同 Cell
```

---

## Q4：@Contended 注解的使用限制有哪些？

**A**：

1. **需 JVM 参数**：用户代码必须添加 `-XX:-RestrictContended` 参数，否则注解不生效。该参数会关闭安全限制，允许用户类使用此内部注解。
2. **包路径**：位于 `jdk.internal.vm.annotation.Contended`，是 JDK 内部 API，非公开 API，未来版本可能变更。
3. **作用范围**：可标注在字段上（仅隔离该字段）或类上（隔离类中所有字段），也可指定分组 `@Contended("group1")`。
4. **内存开销**：会增加对象大小，因为 JVM 会在字段间插入填充字节。

---

## Q5：实际项目中哪些场景容易出现伪共享？

**A**：常见场景：

- **并发计数器**：多个 AtomicLong 紧密排列在数组或对象中
- **Disruptor RingBuffer**：多个生产者/消费者共享环形数组相邻槽位
- **线程状态统计**：线程池中多个线程各自更新统计变量但放在同一对象中
- **数组并行操作**：多线程分别修改数组的相邻元素（如并行排序、并行归约）

实际项目中最有效的做法是优先使用 JDK 提供的并发工具（如 `LongAdder`、`ConcurrentHashMap`），它们内部已经处理了伪共享问题，只有在自定义高性能数据结构时才需要手动处理。

---

## Q6：Disruptor 框架是如何解决伪共享问题的？

**A**：Disruptor 通过**前置+后置填充**确保每个被保护字段独占一个缓存行，是工业级缓存行填充的最佳实践：

```java
// Disruptor 的 Sequence 字段（高并发核心）
abstract class RingBufferPad {
    // 7 个 long 填充 = 56 字节
    protected long p1, p2, p3, p4, p5, p6, p7;
}

abstract class RingBufferFields extends RingBufferPad {
    protected long value;  // 真正的 Sequence（8 字节）
}

class SingleProducerSequencerFields extends RingBufferFields {
    // 同样 7 个 long 填充（后置）
    protected long p1, p2, p3, p4, p5, p6, p7;
}

// 实际占用：7*8 + 8 + 7*8 = 128 字节
// → 远超 64 字节缓存行，字段完全独占一行
```

**Disruptor 的核心设计原则**：
1. **每个原子计数器（Sequence）独占一个缓存行**，生产者/消费者各自独立
2. **前置+后置填充**，防止相邻字段侵入同一缓存行
3. **无锁 + 缓存行填充 + CAS**，实现百万级 QPS（QPS: Queries Per Second）

**实际性能数据**（Disruptor 官方测试）：
- 相比传统队列（有锁）：吞吐量提升 **10~100 倍**
- 在 3.0GHz i7 处理器上达到每秒 **2500 万+** 的事件处理能力

> **面试亮点**：Disruptor 的设计是 JMM 理论（缓存一致性协议、内存屏障）与工程实践完美结合的典范，也是 LMAX 架构的核心。

## 关联知识点
