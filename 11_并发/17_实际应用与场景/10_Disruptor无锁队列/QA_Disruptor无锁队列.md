---
tags: [Java并发, Disruptor, RingBuffer, 无锁队列, 高性能]
module: 17_实际应用与场景
chapter: 10_Disruptor无锁队列
---

# Disruptor无锁队列

## Q1：Disruptor 为什么比 BlockingQueue 性能更好？

**A**：Disruptor 通过多项优化实现超越 BlockingQueue 一个数量级的性能：

| 优化点 | BlockingQueue | Disruptor |
|--------|--------------|-----------| 
| 锁 | synchronized / Lock | 无锁（CAS + Sequence） |
| 内存分配 | 运行时动态创建 | 预分配 Ring Buffer |
| GC | 频繁创建/销毁对象 | 几乎无 GC |
| 伪共享 | 未处理 | 缓存行填充（@Contended） |
| 等待机制 | Object.wait/notify | WaitStrategy 灵活选择 |

LMAX 论文数据：三阶段处理管道，单线程每秒 600 万+ 订单。

---

## Q2：Disruptor 的核心组件有哪些？

**A**：

| 组件 | 职责 |
|------|------|
| **Ring Buffer** | 环形数组，预分配所有槽位 |
| **Sequence** | 无锁原子序号，替代锁 |
| **Sequence Barrier** | 管理消费者间依赖关系 |
| **Event** | 数据载体，存储在 Ring Buffer 槽位 |
| **EventProcessor** | 事件处理器（消费者） |
| **Producer** | 事件发布者 |

核心流程：
```
Producer → CAS 获取 Sequence → 写入 Ring Buffer → publish（设置游标）
Consumer → Barrier 等待可用 Sequence → 处理事件 → 更新消费序号
```

---

## Q3：Disruptor 是如何消除伪共享的？

**A**：通过**缓存行填充**（Cache Line Padding）：

1. CPU 缓存行通常为 64 字节
2. Sequence 的 value（long，8字节）放在缓存行中间
3. 前后各填充 7 个 long（56字节），确保独占一个缓存行
4. 不同线程的 Sequence 不会出现在同一缓存行，消除 MESI 协议的缓存失效开销

```java
// 每个填充字段 8 字节，7个 = 56字节 + value 8字节 = 64字节
class LhsPadding { volatile long p1, p2, p3, p4, p5, p6, p7; }
class Value extends LhsPadding { volatile long value; }
class RhsPadding extends Value { volatile long p8, p9, p10, p11, p12, p13, p14; }
```

---

## Q4：Disruptor 的适用场景和不适用场景？

**A**：

**适用**：
- ✅ **超高性能**：金融交易、订单撮合（600 万+ TPS）
- ✅ **日志框架**：Log4j2 AsyncLogger 使用 Disruptor
- ✅ **事件驱动**：消息分发、事件溯源
- ✅ **单 JVM 内高性能队列**：避免跨进程序列化开销

**不适用**：
- ❌ **低并发场景**：初始化开销大（Ring Buffer 预分配内存），低并发不划算
- ❌ **需要优先级**：Disruptor 是严格 FIFO，不支持优先级排序
- ❌ **动态大小**：Ring Buffer 大小固定，必须预估最大容量
- ❌ **跨进程通信**：Disruptor 是进程内队列，不支持分布式

---

## Q5：单生产者（SINGLE）和多生产者（MULTI）有什么区别？

**A**：

```java
// 单生产者模式（更快，但只能有1个生产者线程）
Disruptor<Event> disruptor = new Disruptor<>(
    Event::new, 1024,
    Executors.defaultThreadFactory(),
    ProducerType.SINGLE,   // ← 单生产者
    new YieldingWaitStrategy()
);

// 多生产者模式（安全，支持多线程并发发布）
Disruptor<Event> disruptor = new Disruptor<>(
    Event::new, 1024,
    Executors.defaultThreadFactory(),
    ProducerType.MULTI,    // ← 多生产者
    new BlockingWaitStrategy()
);
```

| 维度 | SINGLE | MULTI |
|------|--------|-------|
| 发布机制 | 直接更新 cursor（无锁） | 多生产者用 CAS 竞争 slot，bitmap 追踪已发布 |
| 性能 | 更高 | 略低（多一次 CAS 竞争） |
| 线程安全 | ❌ 只有1个生产者才安全 | ✅ 任意多生产者 |
| 适用 | 单线程生产、线程池提交 | 多线程并发发布事件 |

**踩坑**：多生产者场景错用 `SINGLE` → 序号竞争 → 事件覆盖 → 数据丢失。

---

## Q6：Disruptor 的消费者依赖链是怎么工作的？

**A**：

**应用场景**：消费者 C 必须在消费者 A 和 B 都处理完某事件后才能处理（如：先持久化、再发通知）

```java
// 消费者依赖链配置
Disruptor<TradeEvent> disruptor = new Disruptor<>(...);

// 方式1：顺序依赖（A → B → C）
disruptor
    .handleEventsWith(new HandlerA())        // A 先处理
    .then(new HandlerB())                    // B 等 A 完成
    .then(new HandlerC());                   // C 等 B 完成

// 方式2：菱形依赖（A+B 并行，C 等 A+B 都完成）
disruptor
    .handleEventsWith(new HandlerA(), new HandlerB())   // A 和 B 并行
    .then(new HandlerC());                               // C 等 A+B 都完成后执行

// 方式3：广播（A+B+C 各自独立处理同一事件）
disruptor.handleEventsWith(new HandlerA(), new HandlerB(), new HandlerC());
```

**Sequence Barrier 原理**：
- 每个消费者有自己的 Sequence（记录已处理到的序号）
- 依赖链中，后置消费者的 Barrier 会监视前置消费者的 Sequence
- 后置消费者必须等待：`min(前置消费者Sequence) >= 当前要处理的序号`
- 整个过程无锁，通过轮询/等待策略实现

```
生产者游标: 100
HandlerA.sequence: 98   ← A 处理到 98
HandlerB.sequence: 97   ← B 处理到 97
HandlerC.sequence: 96   ← C 等 min(A, B) = 97，所以 C 只能处理到 97
```
