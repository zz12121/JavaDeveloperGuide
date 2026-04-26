---
title: ConcurrentSkipListSet
tags:
  - Java/并发
  - 问答
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentSkipListSet

## Q1：ConcurrentSkipListSet 是怎么实现的？

**A**：底层封装 ConcurrentSkipListMap，所有操作委托给 Map：

```java
private final ConcurrentNavigableMap<E, Object> m;
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return m.putIfAbsent(e, PRESENT) == null;
}
```

key 是元素，value 是固定的 PRESENT 对象。利用 Map 的 key 唯一性实现 Set 去重。

---

## Q2：ConcurrentSkipListSet 和 CopyOnWriteArraySet 怎么选？

**A**：

- **需要有序、元素多**：ConcurrentSkipListSet（O(log n) 操作）
- **元素少、读远多于写**：CopyOnWriteArraySet（读无锁，写复制）
- **需要范围查询**：ConcurrentSkipListSet（支持 subSet、headSet）
- **简单去重**：都可以，小数据量用 COWArraySet 更简单

