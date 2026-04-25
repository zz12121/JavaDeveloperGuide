---
title: volatile应用场景
tags:
  - Java/并发
  - 问答
  - 场景型
module: 03_volatile
created: 2026-04-18
---

# volatile应用场景（状态标志/双重检查锁/单例模式）

## Q1：volatile 有哪些典型的使用场景？

**A**：

1. **状态标志位**：一个线程修改标志，其他线程轮询检查
```java
private volatile boolean running = true;
// 写线程：running = false;
// 读线程：while (running) { ... }
```

2. **DCL 双重检查锁单例**：防止 `new` 操作的指令重排序
```java
private static volatile Singleton instance;
```

3. **一写多读的配置/数据**：保证配置变更对所有线程立即可见

4. **高级模式：开销较低的读-写模式**：读远多于写时避免 synchronized 的互斥开销

---

## Q2：如何判断一个场景是否适合用 volatile？

**A**：两个判断标准：

1. **变量的写入不依赖当前值**：如 `running = false`，而非 `count++`（读-改-写）
2. **变量不参与不变性条件的约束**：如不会同时与其他变量做原子更新

如果两个条件都满足，volatile 就够用；否则需要 `synchronized`、`AtomicXxx` 或 `Lock`。

---

## Q3：DCL 单例中不使用 volatile 会有什么问题？

**A**：`new Singleton()` 在 JVM 中分为三步：① 分配内存 → ② 初始化对象 → ③ 引用指向内存。

没有 volatile 时步骤 ②③ 可能重排为 ①→③→②。线程 A 执行到步骤 ③ 时，线程 B 判断 `instance != null` 就直接返回使用了，但此时对象尚未初始化完成（步骤 ② 还没执行），导致使用了一个半成品对象。volatile 通过内存屏障禁止了这个重排序。

## 关联知识点
