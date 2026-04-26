---
title: CAS原理
tags:
  - Java/并发
  - 问答
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS原理（三个操作数：内存位置V/期望值A/新值B）

## Q1：什么是 CAS？它的工作原理是什么？

**A**：CAS（Compare And Swap）是一种乐观锁实现，包含三个操作数：

- **V**（Value）：内存中变量的实际值
- **A**（Expected）：线程期望的旧值
- **B**（New）：要写入的新值

执行逻辑：当且仅当 V == A 时，才将 V 更新为 B，否则返回失败。整个过程是 CPU 级别的原子指令，不需要加锁。

---

## Q2：CAS 与 synchronized 有什么区别？

**A**：两者本质不同：

| 维度 | CAS | synchronized |
|------|-----|-------------|
| 锁类型 | 乐观锁（无阻塞） | 悲观锁（阻塞式） |
| 原理 | CPU 原子指令 | 对象监视器 |
| 线程阻塞 | 不阻塞，自旋重试 | 阻塞等待 |
| 适用场景 | 竞争不激烈、轻量操作 | 竞争激烈、临界区较大 |
| ABA 问题 | 存在 | 不存在 |

---

## Q3：CAS 如何实现线程安全？举例说明

**A**：以 `AtomicInteger.incrementAndGet()` 为例：

```java
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}

// Unsafe.getAndAddInt 内部实现
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // 读当前值V
    } while (!compareAndSwapInt(o, offset, v, v + delta)); // CAS(V, V, V+1)
    return v;
}
```

1. 读取内存中的当前值 V
2. 计算新值 V+1
3. 执行 CAS(V, V, V+1)：如果 V 没变，更新成功；否则自旋重试

## 关联知识点