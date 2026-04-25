---
title: volatile vs synchronized
tags:
  - Java/并发
  - 问答
  - 对比型
module: 03_volatile
created: 2026-04-18
---

# volatile vs synchronized（volatile轻量，synchronized保证原子性）

## Q1：volatile 和 synchronized 的核心区别？

**A**：

| 维度     | volatile                     | synchronized               |
| -------- | ---------------------------- | -------------------------- |
| 原子性   | ❌ 不保证                     | ✅ 保证                     |
| 可见性   | ✅ 保证                       | ✅ 保证                     |
| 有序性   | ✅ 禁止重排序                 | ✅ 互斥即有序               |
| 阻塞     | 不阻塞                       | 会阻塞                     |
| 开销     | 极低（内存屏障）             | 较高（OS 上下文切换）      |

一句话：**volatile 是轻量级的可见性+有序性保证，synchronized 是重量级的可见性+有序性+原子性保证**。

---

## Q2：什么时候用 volatile，什么时候用 synchronized？

**A**：
- **用 volatile**：状态标志位、DCL 单例、一写多读配置、不涉及复合操作的简单读写
- **用 synchronized**：复合操作（如 i++）、临界区保护、多变量联动更新、需要原子性保证

选择标准：如果变量写入**不依赖当前值**且**不参与不变性条件的约束**，用 volatile；否则用 synchronized。

---

## Q3：volatile 的性能一定比 synchronized 好吗？

**A**：不一定。
- 在**低竞争**场景下，volatile 的内存屏障开销确实远小于 synchronized
- 在**高竞争**场景下，synchronized 有锁升级机制（偏向锁 → 轻量级锁 → 重量级锁），JIT 还可以做锁消除和锁粗化优化，性能未必差
- JDK 6 之后 synchronized 性能大幅优化，不应该仅因性能原因选择 volatile

> **代码示例：volatile 与 synchronized 的典型用法对比**

```java
// ✅ volatile：状态标志位（不依赖当前值的简单读写）
volatile boolean shutdown = false;
void stop() { shutdown = true; }
void work() { while (!shutdown) { doTask(); } }

// ❌ volatile 不能保证原子性的复合操作
volatile int count = 0;
count++; // 实际是 read-modify-write 三步，非原子！

// ✅ synchronized：复合操作需要原子性保证
int count = 0;
synchronized void increment() { count++; }
```

## 关联知识点
