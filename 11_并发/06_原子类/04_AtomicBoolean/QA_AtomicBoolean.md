---
title: AtomicBoolean
tags:
  - Java/并发
  - 问答
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicBoolean（compareAndSet实现DCL单例）

## Q1：AtomicBoolean 最常用的场景是什么？

**A**：通过 `compareAndSet(false, true)` 实现"只执行一次"的语义：

```java
AtomicBoolean initialized = new AtomicBoolean(false);
if (initialized.compareAndSet(false, true)) {
    // 只有一个线程能进入这里
    init();
}
```

典型场景：状态标记、一次性初始化、DCL 单例标志位、任务是否完成的标记。

---

## Q2：AtomicBoolean 的 compareAndSet 和 synchronized 有什么区别？

**A**：`compareAndSet` 不是完整的锁：

| 维度 | compareAndSet | synchronized |
|------|--------------|-------------|
| 粒度 | 只保证一次 CAS 操作 | 保护整个代码块 |
| 阻塞 | 不阻塞，自旋 | 阻塞等待 |
| 可重入 | ❌ 不支持 | ✅ 支持 |
| 异常安全 | 需要手动 reset | 自动释放 |
| 适用场景 | 简单标志位 | 复杂临界区 |

`AtomicBoolean` 只适合"设置一个标记"这种极简场景，复杂逻辑仍需 synchronized 或 Lock。

---

## Q3：用 AtomicBoolean 实现一个简单的互斥锁有什么问题？

**A**：问题在于不可重入和没有条件变量：

```java
// 不可重入！同一线程第二次 lock 会死锁
lock.lock();   // CAS(false, true) ✓
lock.lock();   // CAS(true, true) → 永远失败，死锁！
```

作为"只执行一次"的标记位 AtomicBoolean 足够，但作为通用锁不合适。

## 关联知识点
