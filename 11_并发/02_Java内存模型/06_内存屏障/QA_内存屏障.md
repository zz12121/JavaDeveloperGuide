---
title: 内存屏障
tags:
  - Java/并发
  - 问答
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# 内存屏障（LoadLoad/StoreStore/LoadStore/StoreLoad，JMM通过内存屏障实现可见性和有序性）

## Q1：什么是内存屏障？为什么需要内存屏障？

**A**：内存屏障（Memory Barrier，也称 Memory Fence）是一种 CPU 指令，用于：

1. **禁止特定类型的指令重排序**
2. **强制刷新 CPU 缓存/缓冲区**，保证内存可见性

**为什么需要内存屏障？**

现代 CPU 为了提升性能，采用了多种优化机制：

**① Store Buffer（写缓冲）**
```
CPU 写操作 → 写入 Store Buffer → 异步写回主内存
```
- 优点：CPU 不必等待内存写入完成
- 问题：其他 CPU 可能读到旧值

**② Invalidate Queue（失效队列）**
```
收到缓存失效消息 → 进入队列 → 延迟处理
```
- 优点：CPU 不必立即处理失效
- 问题：可能继续使用过期数据

**内存屏障的作用**：强制刷新这些缓冲区，确保所有 CPU 看到一致的内存状态。

---

## Q2：四种内存屏障类型分别有什么作用？

**A**：JMM 定义了四种基本内存屏障：

| 屏障类型 | 作用 | 典型使用场景 |
|---------|------|-------------|
| **LoadLoad** | 禁止前面的 Load 与后面的 Load 重排序 | volatile 读之后、锁获取 |
| **StoreStore** | 禁止前面的 Store 与后面的 Store 重排序 | volatile 写之前、锁释放 |
| **LoadStore** | 禁止前面的 Load 与后面的 Store 重排序 | volatile 读写之间 |
| **StoreLoad** | 禁止前面的 Store 与后面的 Load 重排序 | volatile 写之后、volatile 读之前 |

**详细解释：**

**LoadLoad 屏障：**
```
Load1;        // 普通读
LoadLoad;     // 屏障
Load2;        // volatile 读 / 锁获取
// Load1 必须完成后，才能执行 Load2
```

**StoreStore 屏障：**
```
Store1;        // 普通写
StoreStore;    // 屏障
Store2;        // volatile 写 / 锁释放
// Store1 必须完成后，才能执行 Store2
```

**LoadStore 屏障：**
```
Load;          // 普通读
LoadStore;     // 屏障
Store;         // 普通写
// Load 必须完成后，才能执行 Store
```

**StoreLoad 屏障（最昂贵）：**
```
Store;         // volatile 写
StoreLoad;     // 屏障
Load;          // volatile 读
// Store 写入主内存后，才能执行 Load
```

---

## Q3：volatile 的内存屏障是如何插入的？

**A**：JVM 为 volatile 操作插入以下屏障：

**volatile 写之前（StoreStore）：**
```java
public void writer() {
    a = 1;                 // 普通写
    StoreStore 屏障;       // 禁止上面的写重排序到 volatile 写之后
    flag = true;           // volatile 写
    StoreLoad 屏障;        // 刷新 Store Buffer
}
```

**volatile 读之后（LoadLoad + LoadStore）：**
```java
public void reader() {
    LoadLoad 屏障;         // 禁止上面的读重排序到 volatile 读之后
    LoadStore 屏障;       // 禁止上面的读重排序到 volatile 写之前
    if (flag) {            // volatile 读
        int i = a;         // 普通读 - 一定能读到 a=1
    }
}
```

**屏障组合效果：**
```
┌──────────────────────────────────────────────────────────┐
│ 线程 A                          线程 B                    │
│ a = 1;                                                     │
│ StoreStore屏障 ────────────→ (a=1 必须在 flag=true 之前完成)│
│ flag = true; (volatile写)                                  │
│ StoreLoad屏障 ─────────────→ (刷新 Store Buffer)          │
│                        ↕ happens-before                    │
│                  if (flag) {                              │
│                  LoadLoad屏障                              │
│                  LoadStore屏障                             │
│                      int i = a;  ← 一定能读到 a=1         │
│                  }                                        │
└──────────────────────────────────────────────────────────┘
```

---

## Q4：为什么 StoreLoad 屏障是最昂贵的？

**A**：StoreLoad 屏障需要：

1. **刷新 Store Buffer**：将所有 pending 的写操作写入主内存
2. **刷新 CPU 缓存**：使其他 CPU 的缓存失效
3. **全局同步**：需要等待所有 CPU 响应失效消息

**x86 架构上的实现：**
```assembly
; StoreLoad 屏障
mfence          ; 或使用 lock 前缀
; 或者
lock add [rsp], 0
```

**性能影响：**
- StoreLoad 是**全屏障**（full barrier）
- 它阻止了几乎所有类型的重排序
- 在 x86 上，由于已经保证了 Store-Load 顺序，StoreLoad 主要用于刷新 Store Buffer

**为什么 volatile 读也需要 StoreLoad？**
```java
if (flag) {        // volatile 读
    int i = a;     // 后续读
}
```
- volatile 读会**消费**之前所有 StoreLoad 屏障的刷新结果
- 确保看到所有 volatile 写之前的普通写操作

---

## Q5：synchronized 的内存屏障是如何实现的？

**A**：synchronized 通过 **monitorenter** 和 **monitorexit** 指令实现锁的获取和释放，JVM 在这些指令周围插入内存屏障。

**monitorenter（获取锁）序列：**
```java
Load obj.header;      // 加载对象头（轻量级锁/偏向锁标记）
LoadLoad 屏障;
LoadStore 屏障;
Store mark;           // 设置为锁定状态
// 成功获取锁
```

**monitorexit（释放锁）序列：**
```java
Store obj.header;     // 标记为无锁状态
StoreStore 屏障;      // 释放前的操作必须先完成
StoreLoad 屏障;      // 刷新 Store Buffer，使其他 CPU 可见
Load obj.header;      // 再次加载（检查偏向锁等）
```

**关键点：**
- **StoreStore 屏障**：保证 unlock 之前的所有操作都已完成
- **StoreLoad 屏障**：保证释放锁后的操作能看到临界区内的所有修改

---

## Q6：内存屏障与 Happens-Before 有什么关系？

**A**：**Happens-Before 是规则，内存屏障是实现**。

**关系图：**
```
┌─────────────────────────────────────────────────────┐
│                 Happens-Before 规则                  │
│                                                      │
│  • 程序次序规则：编译器优化边界                       │
│  • volatile 规则：内存屏障禁止重排序                  │
│  • 管程锁规则：锁获取/释放 的内存屏障                 │
│  • 线程启动/终止：特殊的同步点                        │
└─────────────────────────────────────────────────────┘
                         ↓ 实现机制
┌─────────────────────────────────────────────────────┐
│                   内存屏障                           │
│                                                      │
│  • LoadLoad / StoreStore / LoadStore / StoreLoad    │
│  • 在正确的位置插入正确的屏障                         │
│  • 实现可见性和有序性                                │
└─────────────────────────────────────────────────────┘
```

**举例说明：**

**volatile 规则** → 通过内存屏障实现：
```
volatile 写: StoreStore + StoreLoad 屏障
volatile 读: LoadLoad + LoadStore 屏障
```

**管程锁规则** → 通过内存屏障实现：
```
lock: LoadLoad + LoadStore + 标记锁定
unlock: Store + StoreStore + StoreLoad + Load
```

---

## Q7：x86 架构上内存屏障的实现有什么特点？

**A**：x86 是**强内存模型**，硬件层面已经禁止了大部分重排序：

| 屏障类型 | x86 实现 | 说明 |
|---------|---------|------|
| LoadLoad | `lfence` | 读屏障（x86 上较少使用） |
| StoreStore | `sfence` | 写屏障（x86 上较少使用） |
| LoadStore | `lfence` | 读屏障 |
| StoreLoad | `mfence` 或 `lock prefix` | 全屏障（最常用，也最昂贵） |

**x86 的特点：**
1. **天然保证 Store-Store 顺序**：所有 CPU 看到相同顺序的写
2. **天然保证 Load-Load 顺序**：同一 CPU 的读不会重排序
3. **不保证 Store-Load 顺序**：这是 x86 唯一允许的重排序

**因此：**
- x86 上大部分屏障是**空操作**（no-op）
- **StoreLoad 是唯一真正执行工作的屏障**
- volatile 的主要开销在 StoreLoad 屏障

**JVM 实现策略：**
- 在 x86 上，JVM 可以**省略不必要的屏障**
- 只在关键位置插入 `mfence` 或 `lock` 前缀
- 这使得 volatile 在 x86 上的开销相对较小

---

## Q8：Java 中是否可以手动插入内存屏障？如何操作？

**A**：**可以**，通过 `VarHandle`（Java 9+）或 `Unsafe`（Java 8+）显式插入内存屏障：

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

// 获取 VarHandle
VarHandle vh = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "field", int.class);

// 显式插入屏障
vh.loadLoadFence();    // LoadLoad 屏障
vh.storeStoreFence();  // StoreStore 屏障
vh.loadStoreFence();  // LoadStore 屏障
vh.storeLoadFence();  // StoreLoad 屏障
vh.fullFence();       // 全屏障（最严格）

// Unsafe 层面（Java 8+）
sun.misc.Unsafe unsafe = /* 获取 Unsafe */;
unsafe.loadFence();   // 读屏障
unsafe.storeFence();  // 写屏障
unsafe.fullFence();   // 全屏障
```

**典型应用场景**：
- **Disruptor 环形队列**：生产者/消费者使用 `StoreLoad` 屏障实现高效的无锁队列
- **ConcurrentHashMap**：部分操作使用显式屏障而非 synchronized
- **AQS 框架**：AbstractQueuedSynchronizer 大量使用 `fullFence()` 确保状态修改的原子性

> **注意**：显式屏障是**高级并发编程**工具，普通业务代码直接使用 `volatile` / `synchronized` 即可。除非开发高性能并发库，否则不建议手动插入屏障。

## 关联知识点
