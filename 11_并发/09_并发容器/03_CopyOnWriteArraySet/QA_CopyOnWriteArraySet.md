---
title: CopyOnWriteArraySet
tags:
  - Java/并发
  - 问答
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# CopyOnWriteArraySet

## Q1：CopyOnWriteArraySet 是怎么实现去重的？

**A**：底层基于 CopyOnWriteArrayList，add 操作调用 `addIfAbsent()`：

1. 先 volatile 读数组快照，遍历检查元素是否已存在
2. 存在 → 返回 false
3. 不存在 → 加锁，双重检查确认不存在，复制新数组并添加

```java
public boolean add(E e) {
    return al.addIfAbsent(e);
}
```

---

## Q2：CopyOnWriteArraySet 有什么性能问题？

**A**：

1. **add 是 O(n)**：每次 add 需要遍历数组检查是否存在
2. **contains 是 O(n)**：线性遍历查找
3. **写时复制**：和 COWList 一样有复制开销

如果需要高效的 contains 和 add，应该用 `ConcurrentHashMap.newKeySet()` 或 `Collections.newSetFromMap(new ConcurrentHashMap<>())`。


