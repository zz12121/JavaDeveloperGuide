---
title: TreeMap 底层实现
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-25
---

# TreeMap 底层实现

## Q1：TreeMap 的底层实现是什么？

**A：**
TreeMap 底层是**红黑树（Red-Black Tree）**，一种自平衡的二叉搜索树。每个节点包含 key、value、左右子节点指针、父节点指针和颜色（红/黑）。红黑树通过着色规则保持近似平衡，保证查找、插入、删除的时间复杂度都是 **O(log n)**。

---

## Q2：TreeMap 和 HashMap 的区别？

**A：**

| 维度 | HashMap | TreeMap |
|------|---------|---------|
| 底层结构 | 数组+链表+红黑树 | 红黑树 |
| 遍历顺序 | 无序 | Key 自然排序 |
| 性能 | O(1) 均摊 | O(log n) |
| null key | ✅ 允许 | ❌ 不允许 |
| null value | ✅ 允许 | ✅ 允许 |
| 适用场景 | 普通映射，查找为主 | 需要按 Key 排序、范围查找 |

---

## Q3：TreeMap 是如何保证 Key 有序的？

**A：**
TreeMap 在插入元素时，根据 Key 的 `compareTo()`（或自定义 `Comparator`）比较结果确定节点位置。左子树所有节点都比父节点小，右子树所有节点都比父节点大。中序遍历即为有序序列。

---

## Q4：TreeMap 的红黑树是如何自平衡的？

**A：**
红黑树通过以下机制保持平衡：
1. **着色规则**：节点非红即黑，根节点和叶子节点（NIL）为黑
2. **旋转操作**：插入或删除后，通过左旋或右旋调整结构
3. **最多 2 次旋转**：任何插入或删除操作，最多只需要 2 次旋转就能恢复平衡

---

## Q5：TreeMap 为什么不允许 null key？

**A：**
TreeMap 使用 `compareTo()` 比较 Key 的大小。如果 Key 为 null，调用 `null.compareTo(...)` 会抛出 `NullPointerException`。这与 HashMap（允许一个 null key）不同。

---

## Q6：TreeMap 的 compareTo 和 equals 一致性问题？

**A：**
TreeMap 用 `compareTo()` 判断两个 Key 是否相等（返回 0 表示相等），而 HashMap 用 `equals()` 判断。**两者必须一致**，否则会导致：
- `HashSet` 能去重的对象，`TreeSet` 却无法去重（或反过来）
- `containsKey()` 和 `equals()` 结果不一致

建议：自定义对象同时重写 `equals()` 和 `compareTo()`，保持一致逻辑。

---

## Q7：TreeMap 是线程安全的吗？如何实现线程安全？

**A：**
TreeMap 是**非线程安全**的。多线程环境下可以使用：
1. `Collections.synchronizedSortedMap(new TreeMap<>())` — 包裹一层同步
2. `ConcurrentSkipListMap` — 基于跳表的并发安全 Map，性能更好

---

## Q8：TreeMap 支持哪些导航操作？

**A：**
TreeMap 实现了 `NavigableMap` 接口，提供丰富的导航方法：
- `lowerKey(k)` / `floorKey(k)` / `higherKey(k)` / `ceilingKey(k)` — 小于/小于等于/大于/大于等于
- `firstKey()` / `lastKey()` — 最小/最大 Key
- `subMap(k1, k2)` / `headMap(k)` / `tailMap(k)` — 范围视图
