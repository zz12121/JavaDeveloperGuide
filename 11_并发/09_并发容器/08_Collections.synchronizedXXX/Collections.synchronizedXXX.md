---
title: Collections.synchronizedXXX
tags:
  - Java/并发
  - 对比型
module: 09_并发容器
created: 2026-04-18
---

# Collections.synchronizedXXX

## 核心结论

`Collections.synchronizedXXX` 是一种**同步包装**方式，将普通集合包装为线程安全集合。所有公共方法用 synchronized 修饰（锁为 mutex 对象）。性能较差，推荐优先使用 JUC 并发容器。

## 深度解析

### 原理

```java
// Collections.synchronizedList
public static <T> List<T> synchronizedList(List<T> list) {
    return new SynchronizedList<>(list);
}

static class SynchronizedList<T> extends SynchronizedCollection<T> implements List<T> {
    final List<T> list;
    SynchronizedList(List<T> list) {
        super(list);
        this.list = list;
    }

    // 所有方法都 synchronized
    public E get(int index) {
        synchronized (mutex) { return list.get(index); }
    }
    public E set(int index, E element) {
        synchronized (mutex) { return list.set(index, element); }
    }
    // ...
}
```

### 锁对象

```java
// 默认 mutex 就是集合本身
SynchronizedCollection(Collection<E> c) {
    this.c = Objects.requireNonNull(c);
    this.mutex = this; // mutex = this
}

// 可指定自定义 mutex
SynchronizedCollection(Collection<E> c, Object mutex) {
    this.c = Objects.requireNonNull(c);
    this.mutex = mutex;
}
```

### 可用的包装方法

```java
Collections.synchronizedList(new ArrayList<>());
Collections.synchronizedSet(new HashSet<>());
Collections.synchronizedMap(new HashMap<>());
Collections.synchronizedSortedMap(new TreeMap<>());
Collections.synchronizedSortedSet(new TreeSet<>());
Collections.synchronizedCollection(new ArrayList<>());
Collections.synchronizedNavigableMap(new TreeMap<>());
Collections.synchronizedNavigableSet(new TreeSet<>());
```

### 复合操作不安全

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// 复合操作不安全！需要外部同步
if (!list.contains("key")) {        // step 1: 加锁检查
    list.add("key");                 // step 2: 加锁添加
} // 两步之间可能被其他线程修改

// 正确写法：手动同步
synchronized (list) {
    if (!list.contains("key")) {
        list.add("key");
    }
}
```

### 迭代器不安全

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// 迭代器必须手动同步！
synchronized (list) {
    for (String s : list) {
        // 安全
    }
}

// 不加锁可能抛 ConcurrentModificationException
```

### vs JUC 并发容器

| 维度 | synchronizedXXX | JUC 并发容器 |
|------|----------------|-------------|
| 锁粒度 | 整个集合一把锁 | 细粒度（桶/段/CAS） |
| 并发度 | 低（串行化） | 高 |
| 复合操作 | 不安全 | 提供原子方法 |
| 迭代器 | 需手动同步 | 安全（COW快照/弱一致） |
| 性能 | 低 | 高 |

### vs WeakHashMap（用途区别）

| 维度 | Collections.synchronizedMap | WeakHashMap |
|------|---------------------------|-------------|
| **目的** | 线程安全 | 自动回收没有强引用的 key |
| **线程安全** | ✅ synchronized 包装 | ❌ 非线程安全（需手动包装） |
| **key 回收** | 永不回收（除非显式 remove） | GC 时无强引用则回收 |
| **典型用途** | 一般并发场景 | 缓存（防止内存泄漏） |
| **可组合** | 可用 `Collections.synchronizedMap(new WeakHashMap<>())` | — |

```java
// WeakHashMap：key 无强引用时自动被 GC 回收
WeakHashMap<Key, CacheEntry> cache = new WeakHashMap<>();
// 当 key 不再被其他地方引用，GC 会自动清理

// 组合使用：线程安全 + 弱引用缓存
Map<Key, CacheEntry> safeCache =
    Collections.synchronizedMap(new WeakHashMap<>());
```

**两者用途互补**：synchronizedXXX 解决并发安全，WeakHashMap 解决内存泄漏。

### vs synchronizedNavigableMap vs ConcurrentSkipListMap

| 维度 | Collections.synchronizedNavigableMap | ConcurrentSkipListMap |
|------|--------------------------------------|----------------------|
| 底层 | TreeMap + synchronized | 跳表 + CAS |
| 线程安全方式 | synchronized（单锁） | CAS + volatile（无锁） |
| 有序 | ✅ 自然排序 | ✅ 自然排序 |
| 范围操作 | ✅ subMap/headMap/tailMap | ✅ subMap/headMap/tailMap |
| Navigable API | ✅ lower/ceiling/higher | ✅ lower/ceiling/higher |
| 并发性能 | 低（串行） | 高（无锁） |
| 推荐场景 | 低并发、简单场景 | 高并发、有序 Map |

> ⚠️ `Collections.synchronizedNavigableMap` 底层是 TreeMap，其 `subMap()` 返回的视图是弱一致性的，迭代时也需要外部同步。

## 关联知识点

