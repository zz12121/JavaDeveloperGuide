---
title: HashMap 与 Hashtable 区别
tags:
  - Java/集合框架
  - 对比型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# HashMap 与 Hashtable 区别

## Q1：HashMap 和 Hashtable 的区别？

**A：**

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 不安全 | 安全（synchronized） |
| null key/value | 允许 | 不允许 |
| 默认容量 | 16 | 11 |
| 扩容 | ×2 | ×2+1 |
| 迭代器 | fail-fast | fail-safe |
| 推荐 | ✅ | ❌ |

---

## Q2：Hashtable 的线程安全有什么问题？

**A：**
Hashtable 使用 `synchronized` 修饰所有公共方法，相当于**全表锁**。任何一个操作都会锁住整个 Map，其他线程全部阻塞，并发性能极差。推荐使用 `ConcurrentHashMap` 替代。

---

## Q3：为什么 Hashtable 不允许 null key 和 null value？

**A：**
Hashtable 的 `put()` 方法对 key 和 value 做了 null 检查，为 null 时直接抛 NPE。HashMap 的设计允许 null（null key 的 hash 固定为 0）。这是两个类的设计哲学不同。

---

## Q4：多线程下应该用什么替代 Hashtable？

**A：**
**ConcurrentHashMap**。它使用 CAS + synchronized 锁住单个桶，并发粒度远细于 Hashtable 的全表锁，性能更高。
