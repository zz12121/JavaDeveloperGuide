---
title: volatile可见性原理
tags:
  - Java/并发
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile可见性原理（Store屏障刷新，Load屏障invalidate本地缓存）

## 先说结论

volatile 可见性的本质是通过**内存屏障 + MESI 缓存一致性协议**实现的。写操作后插入 Store 屏障将缓存行刷新到主内存并发出 Invalidate 信号；读操作前插入 Load 屏障使本地缓存行失效，强制从主内存重新加载。在 x86 架构下，由于 x86 本身是强内存模型（TSO），volatile 写只需追加 Lock 前缀即可。

## 深度解析

### JMM 层面：内存屏障

JMM 将内存屏障分为四种：
- **LoadLoad**：确保前面的读在后面的读之前完成
- **LoadStore**：确保前面的读在后面的写之前完成
- **StoreStore**：确保前面的写在后面的写之前完成
- **StoreLoad**：确保前面的写在后面的读之前完成（全能屏障，开销最大）

volatile 写：StoreStore → volatile 写 → StoreLoad
volatile 读：volatile 读 → LoadLoad → LoadStore

### 硬件层面：MESI 协议

CPU 缓存行的四种状态：
- **M（Modified）**：已修改，与主内存不一致，只有当前 CPU 有副本
- **E（Exclusive）**：独占，与主内存一致
- **S（Shared）**：共享，多个 CPU 有副本
- **I（Invalid）**：无效

volatile 写：当前 CPU 缓存行置为 M → 发出 Invalidate → 其他 CPU 置为 I → 刷新到主内存
volatile 读：检查缓存行为 I → 通过总线从主内存加载

### x86 特殊情况

x86 处理器是强内存模型（TSO - Total Store Order），已经保证了：
- Store-Store 顺序（写-写不重排）
- Load-Load 顺序（读-读不重排）
- Load-Store 顺序（读-写不重排）

唯一可能的是 Store-Load 重排序，因此 x86 上的 volatile 写只需追加 **Lock 前缀**（如 `lock addl $0, (%rsp)`）即可，相当于一个 StoreLoad 屏障。

## 易错点/踩坑

- ❌ 以为所有平台上 volatile 的实现都一样
- ✅ x86 只需要 Lock 前缀，ARM/PowerPC 需要完整的内存屏障指令
- ❌ 认为 volatile 读没有任何开销
- ✅ volatile 读需要从主内存重新加载，可能触发 Cache Miss
- ❌ 以为 volatile 的可见性是通过锁实现的
- ✅ volatile 完全无锁，通过缓存一致性协议实现

## 代码示例

```java
// 使用 Unsafe 模拟底层 volatile 语义
public class VolatileUnderlying {
    private volatile int value;

    // JDK 9+ VarHandle 的底层操作
    // VarHandle.ACCESS_MODE.setOpaque()  - 普通写
    // VarHandle.ACCESS_MODE.setVolatile() - volatile 写（带屏障）
    // VarHandle.ACCESS_MODE.getVolatile() - volatile 读（带屏障）
    // VarHandle.ACCESS_MODE.getAcquire()  - acquire 读
    // VarHandle.ACCESS_MODE.setRelease()  - release 写
}
```

## 图解/流程

```
volatile 写的硬件级可见性流程：

  CPU0                            总线                     CPU1
  ┌──────────┐                  ┌──────┐                ┌──────────┐
  │ value=5  │                  │      │                │ value=5  │
  │ (M状态)  │───Invalidate────>│      │───Invalidate──>│ (I状态)  │
  │          │   信号            │      │    信号         │          │
  └──────────┘                  └──────┘                └──────────┘
       │                                                     │
       ▼                                                     ▼
  刷新到主内存                                          下次读取时
  ┌──────────┐                                          ┌──────────┐
  │ 主内存   │                                          │ 从主内存  │
  │ value=6  │                                          │ 重新加载  │
  └──────────┘                                          └──────────┘

x86 平台的 volatile 优化：
  其他平台：StoreStore + volatile写 + StoreLoad（多条屏障指令）
  x86：    lock addl $0, (%rsp) + volatile写（一条指令搞定）
  原因：x86-TSO 已保证其他三种顺序
```

## 关联知识点

