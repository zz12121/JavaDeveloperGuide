---
id: qa_42
title: CAS的CPU支持
tags:
  - Java/并发
  - 问答
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS的CPU支持（X86的CMPXCHG指令，原子完成比较和交换）

## Q1：CAS 底层是由什么指令实现的？

**A**：不同 CPU 架构使用不同指令：

- **X86**：`CMPXCHG`（Compare and Exchange），配合 LOCK 前缀实现原子性
- **ARM**：`LDXR/STXR`（Load-Exclusive / Store-Exclusive）指令对，称为 LL/SC（Load-Linked / Store-Conditional）
- **RISC-V**：`LR/SC`（Load-Reserved / Store-Conditional）

Java 通过 `Unsafe.compareAndSwapInt()` 等 native 方法调用这些底层指令。

---

## Q2：CMPXCHG 如何保证多核原子性？

**A**：通过两种机制：

1. **锁总线**（早期）：CPU 发出 LOCK 信号，锁定前端总线，其他 CPU 无法访问内存。代价大，影响所有内存访问。

2. **缓存锁**（现代，P6 及以后）：利用 MESI 缓存一致性协议，只锁定目标缓存行。其他 CPU 仍可访问其他内存地址。性能远优于锁总线。

```assembly
; X86 汇编
LOCK CMPXCHG [mem], eax
; LOCK 前缀触发缓存锁定机制
```

---

## Q3：为什么说 CAS 比加锁性能好？

**A**：CAS 是用户态操作，不涉及操作系统内核态切换：

| 操作 | CAS | synchronized |
|------|-----|-------------|
| 内核态切换 | ❌ 不需要 | ✅ 需要阻塞时切换 |
| 线程挂起/恢复 | ❌ 不挂起 | ✅ 可能挂起 |
| 上下文切换 | ❌ 无 | ✅ 有 |
| 适用场景 | 短临界区、低竞争 | 长临界区、高竞争 |

但竞争激烈时，CAS 自旋浪费 CPU，性能反而不如阻塞锁。

---

## Q4：什么是 MESI 缓存一致性协议？它和 CAS 有什么关系？

**A**：MESI 是 CPU 多核缓存一致性协议，确保同一缓存行数据在多核间一致，CAS 的原子性依赖它：

```
四种状态：
M (Modified)  — 本核独占，已修改，与主存不一致
E (Exclusive) — 本核独占，与主存一致
S (Shared)    — 多核共享，与主存一致
I (Invalid)   — 缓存行失效
```

**CAS 执行时 MESI 的交互**：

```java
// 线程在 CPU0 执行 CAS(base, 0, 1)
1. CPU0 检测 base 是否在其他核缓存中（总线嗅探）
2. 如果在其他核（S 状态）→ 发送 RFO 令其失效 → CPU0 获得独占权
3. 执行比较和写入（M 状态）
4. 后续 CAS 直接修改，无需总线通信
```

**MESI 的瓶颈**：每次 CAS 都可能触发总线嗅探，高并发下总线带宽成为性能瓶颈。这就是 `LongAdder` 用分段 CAS 减少竞争的原因。

---

## Q5：什么是伪共享？@Contended 注解是如何解决它的？

**A**：伪共享是指两个无关变量恰好在同一个缓存行（64 bytes），一个被频繁修改会导致另一个的缓存行频繁失效：

```java
// ❌ 伪共享：x 和 y 在同一缓存行
class NoPadding {
    volatile long x;
    volatile long y;
}
// 线程1 CAS x，线程2 CAS y
// → x 修改后 y 的缓存行失效
// → 线程2 必须重新加载 y，性能骤降
```

**@Contended 解决**：

```java
// ✅ 每个变量独占缓存行
class WithPadding {
    @jdk.internal.vm.annotation.Contended
    volatile long x;
    @jdk.internal.vm.annotation.Contended
    volatile long y;
}
// 编译器自动在 x 和 y 之间填充 128 bytes
// 线程1 CAS x，线程2 CAS y → 互不干扰
```

**LongAdder.Cell 使用 @Contended 的原因**：高并发下所有 Cell 的 value 被同时 CAS，如果不填充，每个 Cell 独占一个缓存行，线程间 CAS 操作互不干扰。

---

## 关联知识点

