---
title: HashSet 底层实现
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# HashSet 底层实现

## Q1：HashSet 的底层实现是什么？

**A：**
HashSet 底层就是一个 **HashMap**。所有元素存储为 HashMap 的 key，value 统一是一个名为 `PRESENT` 的空 Object 对象。`add(e)` 实际调用 `map.put(e, PRESENT)`，`contains(e)` 调用 `map.containsKey(e)`。

---

## Q2：HashSet 是如何保证元素不重复的？

**A：**
通过 HashMap 的 key 唯一性保证。添加元素时：
1. 先计算 `hashCode()`，定位到桶
2. 如果桶中已有元素，遍历比较 `hashCode()` 和 `equals()`
3. 只有当 `hashCode()` 相同且 `equals()` 返回 true 时，才认为是重复元素

所以自定义对象放入 HashSet 必须正确重写 `hashCode()` 和 `equals()`。

---

## Q3：HashSet 允许 null 吗？

**A：**
允许，最多一个。因为底层 HashMap 允许一个 null key，`put(null, PRESENT)` 只能成功一次。
