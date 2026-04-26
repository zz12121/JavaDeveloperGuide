---
title: AtomicMarkableReference
tags:
  - Java/并发
  - 问答
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicMarkableReference（带标记位的对象引用）

## Q1：AtomicMarkableReference 和 AtomicStampedReference 有什么区别？

**A**：

| 维度 | AtomicStampedReference | AtomicMarkableReference |
|------|----------------------|------------------------|
| 附加信息类型 | int（版本号） | boolean（标记位） |
| 状态数 | 2^32 种 | 2 种 |
| 用途 | 追踪修改次数 | 标记"是否被修改" |
| 适用场景 | 严格版本控制 | GC标记、节点删除标记 |

简单说：**Stamped 追踪"改了几次"，Markable 追踪"改没改过"**。

---

## Q2：什么场景适合用 AtomicMarkableReference？

**A**：适合只需要二元状态标记的场景：

1. **标记删除**：链表/树节点先标记 `mark=true`，再 CAS 移除
2. **GC 标记**：标记对象是否可达
3. **脏标记**：缓存行是否被修改
4. **一次性操作**：标记是否已初始化

```java
// 一次性初始化
AtomicMarkableReference<Config> configRef =
    new AtomicMarkableReference<>(null, false);

boolean[] mark = {false};
Config config = configRef.get(mark);
if (!mark[0]) {
    Config newConfig = loadConfig();
    configRef.compareAndSet(null, newConfig, false, true);
}
```

---

## Q3：标记删除是如何工作的？

**A**：分两步实现安全的节点移除：

```java
// 1. 先标记 mark=true（逻辑删除）
node.next.compareAndSet(succ, succ, false, true);

// 2. 再 CAS 移除前驱节点的 next 指针（物理删除）
prev.next.compareAndSet(node, node.next.getReference(), false, false);
```

标记删除保证：即使并发情况下，其他线程看到 `mark=true` 也能正确处理，不会误操作。

## 关联知识点

