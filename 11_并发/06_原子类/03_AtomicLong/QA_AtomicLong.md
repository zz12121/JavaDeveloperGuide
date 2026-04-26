---
title: AtomicLong
tags:
  - Java/并发
  - 问答
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicLong（64位原子操作，解决Long型读写的线程安全问题）

## Q1：为什么 32 位 JVM 上 long 的读写不是原子的？

**A**：32 位 CPU 一次只能操作 32 位（4 字节），而 long 占 64 位（8 字节）。JVM 写一个 long 需要两条 32 位指令：

```
写 long value = 0x00000000_FFFFFFFF
指令1: 写高 32 位 = 0x00000000
指令2: 写低 32 位 = 0xFFFFFFFF
                    ↑ 线程切换可能发生在这两条指令之间
```

如果在两条指令之间发生线程切换，其他线程可能读到"半新半旧"的值。

`AtomicLong` 通过 `volatile`（可见性）+ `CAS`（原子更新）解决此问题。

---

## Q2：AtomicLong 和 volatile long 有什么区别？

**A**：

- `volatile long`：只保证读写的可见性（每次读到最新值），但 `i++`（读-改-写）仍不是原子的
- `AtomicLong`：既保证可见性（volatile），又保证读-改-写的原子性（CAS）

```java
// volatile long — i++ 仍不安全
volatile long count = 0;
count++; // 实际是 读→+1→写，非原子

// AtomicLong — i++ 安全
AtomicLong count = new AtomicLong(0);
count.incrementAndGet(); // 原子操作
```

---

## Q3：高并发场景下 AtomicLong 有什么问题？

**A**：所有线程竞争同一个 value，高并发下 CAS 失败率高，大量 CPU 空转。推荐用 `LongAdder` 替代：

- `AtomicLong`：适合低并发、需要精确值、需要 compareAndSet 的场景
- `LongAdder`：适合高并发统计场景（QPS、计数），性能好 5~10 倍

## 关联知识点
