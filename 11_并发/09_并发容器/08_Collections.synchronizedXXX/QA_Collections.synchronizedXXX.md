---
title: Collections.synchronizedXXX
tags:
  - Java/并发
  - 问答
  - 对比型
module: 09_并发容器
created: 2026-04-18
---

# Collections.synchronizedXXX

## Q1：Collections.synchronizedXXX 的原理是什么？

**A**：用装饰器模式包装原始集合，所有公共方法加 synchronized（锁为 mutex 对象，默认是集合本身）：

```java
public E get(int index) {
    synchronized (mutex) { return list.get(index); }
}
```

整个集合用同一把锁，所有操作串行化。

---

## Q2：为什么推荐用 JUC 并发容器替代 synchronizedXXX？

**A**：

| 问题 | synchronizedXXX | JUC 并发容器 |
|------|----------------|-------------|
| 锁粒度 | 整个集合一把锁 | 细粒度 |
| 并发度 | 低 | 高 |
| 复合操作 | 不安全 | putIfAbsent 等原子方法 |
| 迭代器 | 需手动加锁 | 安全 |
| 性能 | 差 | 好 |

JUC 并发容器在设计上就是为高并发优化的，synchronizedXXX 只是简单加锁包装。

---

## Q3：synchronizedXXX 的复合操作为什么不安全？

**A**：虽然每个单独方法（contains、add）是线程安全的，但**两个方法之间的组合不是原子的**：

```java
// 不安全
if (!list.contains("key")) {  // 加锁 → 释放锁
    list.add("key");           // 加锁 → 释放锁
} // 两步之间，其他线程可能已经 add 了
```

解决方案：手动 synchronized(list) 包裹复合操作，或使用 JUC 的原子方法（如 putIfAbsent）。

---

## Q4：WeakHashMap 和 Collections.synchronizedMap 有什么区别？

**A**：两者解决的是不同问题：

| 维度 | Collections.synchronizedMap | WeakHashMap |
|------|---------------------------|-------------|
| **目的** | 保证并发访问的线程安全 | 在 key 没有强引用时自动回收 entry |
| **线程安全** | ✅（synchronized 包装） | ❌（需手动包装） |
| **回收机制** | 永不自动回收 | GC 时无强引用 → 自动移除 |
| **key 类型** | 任意对象 | 只能是弱引用（WeakReference） |
| **底层** | 装饰器模式 + synchronized | ReferenceQueue + WeakReference |

```java
// WeakHashMap：适合缓存——key 不再使用时自动清理
WeakHashMap<Key, Value> cache = new WeakHashMap<>();
cache.put(key, value);  // key 无其他引用时，GC 清理

// 线程安全的 WeakHashMap（组合使用）
Map<Key, Value> safeCache =
    Collections.synchronizedMap(new WeakHashMap<>());

// Collections.synchronizedMap：适合一般并发场景
Map<Key, Value> safeMap =
    Collections.synchronizedMap(new HashMap<>());
```

**组合使用**：`Collections.synchronizedMap(new WeakHashMap<>())` 可以同时满足线程安全和自动回收，但实际场景中更推荐使用 `WeakHashMap` 本身 + 手动同步关键操作。

---

## Q5：Collections.synchronizedNavigableMap 和 ConcurrentSkipListMap 怎么选？

**A**：两者都提供有序 NavigableMap 功能，区别在于并发性能：

```java
// 低并发、简单场景 → Collections.synchronizedNavigableMap
NavigableMap<String, Integer> map1 =
    Collections.synchronizedNavigableMap(new TreeMap<>());
// subMap、lower/ceiling 等操作均需外部同步

// 高并发、有序 Map → ConcurrentSkipListMap
ConcurrentSkipListMap<String, Integer> map2 = new ConcurrentSkipListMap<>();
// 所有操作线程安全，无锁并发，NavigableMap 接口完整实现
```

两者 Navigable API 完全一致（lower/ceiling/higher/subMap 等），选型关键：
- **并发度低** → `synchronizedNavigableMap`（简单）
- **高并发** → `ConcurrentSkipListMap`（性能好）


