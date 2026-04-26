---
title: ConcurrentSkipListSet
tags:
  - Java/并发
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentSkipListSet

## 核心结论

ConcurrentSkipListSet 基于 ConcurrentSkipListMap 实现，value 为固定的 Boolean PRESENT。提供线程安全的有序 Set，所有操作平均 O(log n)。

## 深度解析

### 源码

```java
public class ConcurrentSkipListSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {

    private final ConcurrentNavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public ConcurrentSkipListSet() {
        m = new ConcurrentSkipListMap<E, Object>();
    }

    public boolean add(E e) {
        return m.putIfAbsent(e, PRESENT) == null; // key 不存在才添加
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }

    public boolean remove(Object o) {
        return m.remove(o, PRESENT);
    }
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 底层 | ConcurrentSkipListMap |
| 有序 | 自然排序（或 Comparator） |
| 去重 | putIfAbsent |
| 操作复杂度 | O(log n) |
| 范围操作 | 支持（subSet、headSet、tailSet） |
| 线程安全 | CAS + volatile |

### vs CopyOnWriteArraySet

| 维度 | ConcurrentSkipListSet | CopyOnWriteArraySet |
|------|----------------------|---------------------|
| 底层 | 跳表 | 数组 |
| add | O(log n) | O(n) |
| contains | O(log n) | O(n) |
| 有序 | 是 | 否 |
| 写性能 | 好（CAS） | 差（复制数组） |
| 内存 | 索引开销 | 复制开销 |

## 关联知识点
