---
title: 缓存一致性协议
tags:
  - Java/并发
  - 问答
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# 缓存一致性协议（MESI/MESI-Ownership/MOESI）

## Q1：为什么多核 CPU 需要缓存一致性协议？

**A**：现代 CPU 每个核心都有私有 L1/L2 缓存。当多个核心缓存同一内存地址时，如果一个核心修改了数据，其他核心的缓存副本就过时了（Stale Data）。如果没有一致性协议，就会出现数据不一致问题：

```
Core 0: 读 X → 缓存 x=1
Core 1: 读 X → 缓存 x=1
Core 0: 写 X=2 → 本地缓存 x=2（主存未更新）
Core 1: 读 X → 返回缓存中的 x=1 ❌（应该是 2）
```

缓存一致性协议通过**总线监听**（Bus Snooping）或**基于目录**（Directory-based）机制，在核心间传播失效/更新消息，确保所有核心看到一致的值。

---

## Q2：详细描述 MESI 协议的四种状态及其转换条件？

**A**：

**四种状态**：

| 状态 | 含义 | 能否读 | 能否写 | 是否脏 |
|------|------|:---:|:---:|:---:|
| **M** Modified | 本 Core 独有，已修改 | ✅ | ✅ | 是 |
| **E** Exclusive | 本 Core 独有，未修改 | ✅ | ✅ | 否 |
| **S** Shared | 多 Core 共有，只读 | ✅ | ❌ | 否 |
| **I** Invalid | 数据无效 | ❌ | ❌ | — |

**关键转换**：

- **E → M**：本地写。独占状态下写，无需总线消息，最快路径。
- **S → M**：本地写 + 发送 Invalidate + 等待所有 Ack。必须确保其他 Core 的副本失效。
- **E → S**：其他 Core 请求读同一地址。本 Core 降级为 Shared。
- **M → I**：其他 Core 请求读/写同一地址。本 Core 先写回内存再失效。
- **I → E/S**：Cache Miss 后发起 Read，根据其他 Core 是否有副本决定进入 E（无副本）或 S（有副本）。
- **S → I**：其他 Core 写同一地址，本 Core 收到 Invalidate 消息后失效。

---

## Q3：MESI 写操作为什么需要等待 Invalidate Ack？这带来了什么问题？

**A**：当缓存行处于 S（Shared）状态时，多个核心都有只读副本。某 Core 要写时，必须先让所有其他持有该副本的 Core 将其标记为 Invalid（发送 Invalidate 消息），并等待所有 Ack 确认后，才能安全地将状态升级为 M 并执行写操作。

**带来的问题**：

1. **写延迟大**：等待所有 Ack 需要跨越总线的往返时间（数十到上百个时钟周期）
2. **阻塞 CPU 流水线**：写操作必须等待一致性协议完成才能提交，浪费 CPU 周期

**解决方案——Store Buffer**：

CPU 引入了 Store Buffer（写缓冲），核心将写操作先放入 Buffer 立即继续执行后续指令，由一致性协议在后台处理失效+等待。但这引入了新的问题：**Store Forwarding 导致的内存重排序**（写操作的完成顺序可能与程序顺序不同），这也是需要 Memory Barrier 的根本原因。

---

## Q4：MESI、MESI-Ownership、MOESI 三者有什么区别？

**A**：

| 特性 | MESI | MESI-Ownership | MOESI |
|------|------|----------------|-------|
| 状态数 | 4 | 4（含 Owner） | 5（含 Owned） |
| 脏数据能否共享 | ❌ 不能 | ✅ 可以（Owner 转发） | ✅ 可以（Owned） |
| 写回次数 | 较多 | 较少（转发替代写回） | 较少 |
| 代表架构 | Intel/ARM | 部分定制 | AMD |

**关键区别**：

- **MESI**：M 状态的缓存行必须独占。当其他 Core 请求读时，必须先写回内存再失效。
- **MESI-Ownership / MOESI**：引入 Owner/Owned 状态，允许持有脏数据的 Core 直接将最新数据转发给请求者，无需先写回内存，减少了一次总线事务。这对"读多写一"的共享模式性能提升明显。

---

## Q5：缓存一致性协议与 Java volatile 有什么关系？

**A**：Java volatile 的可见性保证底层就依赖 CPU 缓存一致性协议：

| Java volatile 操作 | 对应底层机制 |
|-------------------|-------------|
| volatile 写 | 修改本地缓存 → 发送 Invalidate 消息 → 等待 Ack |
| volatile 读 | 检查本地缓存状态 → 若 Invalid 则发起总线 Read → 获取最新缓存行 |
| happens-before 保证 | Store Buffer + Memory Barrier 确保 Invalidate 消息在后续读之前完成 |

简言之，volatile 写通过触发 MESI 的 Invalidate 流程让其他 Core 的缓存副本失效，volatile 读通过 Cache Miss 强制从总线获取最新值。Memory Barrier（`lock` 前缀指令或 `mfence`）则确保了操作的有序性。

---

## Q6：什么是 Store Buffer 和 Invalidate Queue？它们解决了什么问题又引入了什么问题？

**A**：

**Store Buffer**：
- **解决的问题**：MESI 写等待 Invalidate Ack 太慢，阻塞 CPU
- **机制**：写操作先入 Buffer，CPU 继续执行，后台等待 Ack 后真正写入缓存
- **引入的问题**：后续读操作可能读到 Buffer 中还未提交的旧值（Store Forwarding）；其他 Core 的写可能在本 Core 的写之前完成（写重排序）

**Invalidate Queue**：
- **解决的问题**：接收 Invalidate 消息的 Core 处理太慢（需清空流水线），影响发送方
- **机制**：接收方将 Invalidate 消息暂存到 Queue，后续再处理
- **引入的问题**：接收方可能在 Invalidate 尚未处理时就执行读操作，读到过期数据

这两个优化合在一起使得 CPU 必须提供 **Memory Barrier** 指令，让程序员在需要时手动刷新 Store Buffer、排空 Invalidate Queue，这正是 volatile 和 final 字段底层依赖的机制。

---

## Q7：什么是总线风暴？volatile 滥用会导致什么性能问题？

**A**：**总线风暴（Bus Storm）**是指大量线程频繁对同一 `volatile` 变量执行写操作，导致 MESI 协议的 Invalidate 消息在总线上大量广播，占满总线带宽，使整体性能急剧下降的现象。

**形成过程**：

```java
// 问题代码
private volatile long counter = 0;

public void increment() {
    counter++;  // 每个写都触发 MESI Invalidate + 总线广播
}
```

1. 线程 A 在核心 0 写入 `counter` → 发送 Invalidate 广播到所有核心
2. 线程 B 在核心 1 写入 `counter` → 再次广播 Invalidate
3. 线程 C、D、E……所有核心收到失效消息后又要失效缓存行
4. N 个核心并发写 → **N 次总线广播/秒** → 总线饱和 → 性能崩溃

**性能对比**：

| 方案 | 机制 | QPS（示例） |
|------|------|------------|
| `AtomicLong`（volatile 写） | 每次写触发全局失效广播 | ~10 万 |
| `LongAdder` | 写分散到不同缓存行，减少广播 | ~1000 万 |
| `synchronized` | 互斥锁，高并发下大量线程阻塞 | ~50 万 |

**解决方案**：

```java
// ✅ 方案1：LongAdder 替代 AtomicLong（推荐）
LongAdder counter = new LongAdder();  // 内部 Cell[]，每个 Cell 独立缓存行
counter.increment();  // 写分散，无全局广播

// ✅ 方案2：批量更新减少 volatile 写入频率
private volatile long lastUpdate = 0;
private long localCounter = 0;

public void batchIncrement() {
    localCounter += 1000;  // 本地批量累加
    if (System.currentTimeMillis() - lastUpdate > 1000) {
        lastUpdate = localCounter;  // 每秒只写一次 volatile
    }
}
```

> **面试结论**：volatile 不是万能的，在高并发写同一变量时，Inavlidate 广播会形成性能瓶颈。正确选择 `LongAdder` / 分段锁 / 批量更新等方案。

## 关联知识点

