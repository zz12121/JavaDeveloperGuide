---
title: HashMap 与 Hashtable 的区别
tags:
  - Java/集合框架
  - 对比型
module: 05_集合框架
created: 2026-04-18
---

# HashMap 与 Hashtable 的区别

## 全面对比

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 出现版本 | JDK 1.2 | JDK 1.0（古老） |
| 线程安全 | ❌ 不安全 | ✅ 安全（`synchronized`） |
| null key | ✅ 允许 1 个 | ❌ 不允许（抛 NPE） |
| null value | ✅ 允许 | ❌ 不允许（抛 NPE） |
| 默认初始容量 | 16 | 11 |
| 扩容倍数 | 2 倍 | 2 倍 + 1 |
| 负载因子 | 0.75 | 0.75 |
| 迭代器 | **fail-fast**（Iterator） | **fail-safe**（Enumeration） |
| 父类 | `AbstractMap` | `Dictionary` |
| 是否推荐 | ✅ 推荐 | ❌ 已过时 |

## 线程安全的区别

### Hashtable（全表锁）

```java
// 所有方法都加 synchronized，锁住整个对象
public synchronized V put(K key, V value) { ... }
public synchronized V get(Object key) { ... }
public synchronized V remove(Object key) { ... }
```

- 任何线程操作时，其他线程全部阻塞
- 多线程竞争激烈时性能极差

### HashMap（非线程安全）

```java
// 没有任何同步措施
public V put(K key, V value) { ... }
public V get(Object key) { ... }
```

- 单线程性能最好
- 多线程下可能出现数据不一致、死循环（JDK 7 扩容）等问题

## null 值的区别

```java
// HashMap 允许 null
HashMap<String, String> map = new HashMap<>();
map.put(null, "value");   // ✅
map.put("key", null);     // ✅

// Hashtable 不允许 null
Hashtable<String, String> table = new Hashtable<>();
table.put(null, "value"); // ❌ NullPointerException
table.put("key", null);   // ❌ NullPointerException
```

Hashtable 的 `put()` 方法会对 value 做 `null` 检查，如果 value 或 key 为 null 直接抛 NPE。

## 扩容的区别

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 默认容量 | 16（2 的幂次） | 11（质数） |
| 扩容后容量 | oldCapacity × 2 | oldCapacity × 2 + 1 |
| 容量要求 | 2 的幂次（位运算优化） | 2 × 旧值 + 1（取模） |

HashMap 要求容量为 2 的幂次，这样可以用 `hash & (n - 1)` 代替 `hash % n`，提升性能。

## 替代方案

Hashtable 已过时，推荐替代方案：

| 场景 | 推荐 |
|------|------|
| 单线程 | **HashMap** |
| 多线程 | **ConcurrentHashMap** |
| 需要同步包装 | `Collections.synchronizedMap(new HashMap<>())` |

## 关联知识点

