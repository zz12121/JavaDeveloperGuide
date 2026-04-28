---
title: synchronized 面试题精炼
tags:
  - Java/并发
  - 问答
module: 04_synchronized
created: 2026-04-28
difficulty: 难
---

# synchronized 面试题精炼

> 💡 本文档只讲"怎么答"，原理细节看 [[01_synchronized原理与使用]] 和 [[02_锁升级全过程]]。

---

## Q1：synchronized 和 ReentrantLock 有什么区别？

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 锁获取 | 自动（JVM管理） | 手动（必须 finally unlock） |
| 可中断 | ❌ 不可中断 | ✅ lockInterruptibly() |
| 超时获取 | ❌ | ✅ tryLock(timeout) |
| 公平锁 | ❌ 非公平 | ✅ 可选公平/非公平 |
| 条件变量 | 单个（wait/notify） | 多个（newCondition） |
| 底层实现 | 对象头 + ObjectMonitor | AQS + CAS |
| 性能 | JDK 6+ 优化后差距不大 | 极高并发时略优 |

**一句话**：synchronized 简单安全（不怕忘释放），ReentrantLock 灵活强大（超时/中断/多条件）。

---

## Q2：synchronized 锁升级过程？

**四步，单向不可逆**：无锁(01) → 偏向锁(01/1) → 轻量级锁(00) → 重量级锁(10)

- **偏向锁**：同一线程重入只需比对线程ID，零开销。JDK 15 废弃
- **轻量级锁**：CAS + 自旋，适用于短暂竞争。自旋在用户态完成
- **重量级锁**：ObjectMonitor + OS互斥，竞争线程 BLOCKED。涉及用户态→内核态切换

**JDK 18+ 主流路径**：无锁 → 轻量级锁 → 重量级锁（跳过偏向锁）

---

## Q3：为什么锁升级不可逆？

为了**简化实现 + 避免性能抖动**。如果允许降级，运行时需要不断判断是否该降级，增加复杂性。而且升级本身就意味着曾经竞争过，降级后可能很快又升级，得不偿失。

---

## Q4：偏向锁为什么被废弃？

1. **撤销代价高**：需要在 safepoint 暂停所有线程（STW）
2. **边际收益小**：现代轻量级锁已经够快
3. **代码复杂度**：偏向锁 bug 占 JVM 同步代码很大比例
4. JEP 374 废弃(JDK 15) → JEP 374 移除(JDK 18)

---

## Q5：什么是自适应自旋？和固定自旋的区别？

- **固定自旋**：自旋 N 次后直接膨胀（`-XX:PreBlockSpin=10`）
- **自适应自旋**：JVM 根据历史动态调整次数
  - 上次成功 → 增加次数
  - 上次失败 → 减少甚至不自旋

---

## Q6：什么场景下轻量级锁会膨胀为重量级锁？

1. 自旋超过自适应阈值
2. 竞争线程 ≥ 2
3. 持有锁线程调用了 `wait()`（需要 ObjectMonitor）

---

## Q7：ObjectMonitor 的 _EntryList 和 _WaitSet 区别？

| | _EntryList | _WaitSet |
|--|-----------|----------|
| 线程状态 | BLOCKED | WAITING |
| 进入原因 | 竞争锁失败 | 调用 wait() |
| 离开方式 | 锁释放后被唤醒竞争 | 被 notify/notifyAll 移到 EntryList |

**关键**：notify() 不会直接让线程获取锁，而是从 WaitSet 移到 EntryList 重新竞争！

---

## Q8：synchronized 修饰方法和代码块有什么区别？

```java
// 修饰实例方法 → 锁对象是 this
public synchronized void method() { }

// 修饰静态方法 → 锁对象是 Class 对象
public static synchronized void method() { }

// 修饰代码块 → 锁对象自己指定
synchronized(obj) { }
```

---

## Q9：锁消除和锁粗化是什么？

- **锁消除**：JIT 通过逃逸分析发现对象不会逃逸出线程，自动消除同步（如 StringBuilder 的 append）
- **锁粗化**：JIT 将多次相邻的加锁/解锁合并为一次，减少锁操作次数

```java
// 锁粗化前
synchronized(lock) { a++; }
synchronized(lock) { b++; }
synchronized(lock) { c++; }
// JIT 优化后
synchronized(lock) { a++; b++; c++; }
```

---

## Q10：为什么调用 hashCode() 会导致偏向锁撤销？

Mark Word 空间有限，无锁状态存 hashCode（31bit），偏向锁状态存 threadID（54bit），两者不能共存。调 hashCode() 后 Mark Word 被写入哈希值，偏向锁立即撤销。

---

## 常见错误答法

| ❌ 错误 | ✅ 正确 |
|---------|---------|
| "锁可以降级" | 锁升级**单向不可逆** |
| "偏向锁适用于所有场景" | 偏向锁只适合**几乎无竞争**的场景 |
| "重量级锁一定比轻量级锁慢" | 无竞争时开销接近，激烈竞争时重量级锁反而合理 |
| "notify() 直接唤醒线程获取锁" | notify() 移到 EntryList，仍需**重新竞争锁** |
| "自旋越多越好" | 激烈竞争时自旋浪费 CPU，该膨胀就膨胀 |

---

## 我的回答模板

> ⚡ 面试前只看这个区域
>
> Q: "说一下synchronized锁升级"
> 
> 我的回答："synchronized锁有四个级别——无锁、偏向锁、轻量级锁、重量级锁。JVM根据竞争情况自动升级，单向不可逆。偏向锁在对象头记录线程ID，同一线程重入零开销，但JDK15废弃了。轻量级锁通过CAS和自旋在用户态解决短暂竞争。自旋失败后膨胀为重量级锁，依赖ObjectMonitor和OS互斥，线程进入BLOCKED状态。JDK18之后主流路径是无锁直接到轻量级锁再到重量级锁。"
