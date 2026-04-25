---
title: volatile写读流程
tags:
  - Java/并发
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile写读流程（写后强制刷新主存，读前invalidate CPU缓存）

## 先说结论

volatile 的写操作会触发 **StoreStore + StoreLoad** 内存屏障，强制将工作内存中的值刷新到主内存；volatile 的读操作会触发 **LoadLoad + LoadStore** 内存屏障，使本地 CPU 缓存中的该变量失效，强制从主内存重新加载。这是 volatile 可见性的底层实现机制。

## 深度解析

### volatile 写流程

1. 修改工作内存（寄存器/L1 Cache）中的 volatile 变量
2. 插入 **StoreStore 屏障**：确保之前的普通写操作已经刷新到主内存
3. 将 volatile 变量写回主内存
4. 插入 **StoreLoad 屏障**：确保 volatile 写对后续的读操作可见（最昂贵的屏障）

### volatile 读流程

1. 插入 **LoadLoad 屏障**：确保 volatile 读之前的读操作已经完成
2. 从主内存读取 volatile 变量的最新值（invalidate 本地缓存行）
3. 插入 **LoadStore 屏障**：确保 volatile 读之后的写操作不会被重排到读之前

### 内存屏障与缓存一致性协议的配合

volatile 的内存屏障最终通过底层硬件的 **MESI 缓存一致性协议** 实现：
- **写操作**：CPU 发出 Invalidate 消息，其他 CPU 的缓存行失效，然后写入主内存
- **读操作**：CPU 检查本地缓存行状态，如果为 Invalid 则从主内存重新加载

## 易错点/踩坑

- ❌ 认为 volatile 读写的开销和普通变量一样
- ✅ volatile 写有 StoreLoad 屏障，是最昂贵的内存屏障（近似于一个完整 fence）
- ❌ 以为 volatile 只影响被修饰的变量本身
- ✅ volatile 内存屏障会影响屏障前后的其他内存操作

## 代码示例

```java
public class VolatileReadWriteDemo {
    private volatile int config = 0;

    // 写线程
    public void updateConfig(int newValue) {
        // 普通写1（任意顺序）
        someOrdinaryWrite1();
        // StoreStore 屏障：确保上面的写已刷新
        // ---- volatile 写 ----
        config = newValue;  // 强制刷新到主内存
        // StoreLoad 屏障：确保 volatile 写对后续读可见
        // 普通写2（不能重排到 volatile 写之前）
        someOrdinaryWrite2();
    }

    // 读线程
    public int readConfig() {
        // 普通读1（任意顺序）
        someOrdinaryRead1();
        // LoadLoad 屏障：确保上面的读已完成
        // ---- volatile 读 ----
        int value = config;  // 强制从主内存加载
        // LoadStore 屏障：确保后续写不会重排到读之前
        // 普通操作（不能重排到 volatile 读之前）
        someOrdinaryOperation();
        return value;
    }
}
```

## 图解/流程

```
volatile 写操作：
┌──────────────────────────────────────────────────┐
│ 普通写1    普通写2    │ StoreStore │ volatile写 │ StoreLoad │ 普通操作3   │
│  ↓           ↓        │   屏障     │   ↓         │   屏障    │   ↓         │
│ 刷新到主存  刷新到主存  │ ───────── │ 写入主存    │ ────────  │             │
└──────────────────────────────────────────────────┘

volatile 读操作：
┌──────────────────────────────────────────────────┐
│ 普通读1    │ LoadLoad │ volatile读 │ LoadStore │ 普通操作3   │
│  ↓         │   屏障    │   ↓         │   屏障    │   ↓         │
│            │ ───────── │ 从主存加载  │ ────────  │             │
│            │           │ (缓存失效)  │           │             │
└──────────────────────────────────────────────────┘

跨线程可见性：
  线程A                    主内存                    线程B
  ┌──────┐               ┌──────┐               ┌──────┐
  │ 写操作│ ══刷新═══>   │ val=5 │ ══失效══>   │ 缓存行│
  │      │              │       │  Invalidate   │ I状态 │
  └──────┘              └──────┘              └──────┘
                                                      │
                                               下次读取时
                                                      │
                                               ┌──────┐
                                               │ 从主存│
                                               │ 加载  │
                                               └──────┘
```

## 关联知识点
