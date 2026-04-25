---
title: HashSet 底层实现
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-25
---

# HashSet 底层实现

## Q1：HashSet 的底层实现是什么？

**A：**
HashSet 底层就是一个 **HashMap**。所有元素存储为 HashMap 的 key，value 统一是一个名为 `PRESENT` 的空 Object 对象。`add(e)` 实际调用 `map.put(e, PRESENT)`，`contains(e)` 调用 `map.containsKey(e)`，`remove(e)` 调用 `map.remove(e)`。

---

## Q2：HashSet 是如何保证元素不重复的？

**A：**
通过 HashMap 的 key 唯一性保证。添加元素时：
1. 先计算 `hashCode()`，定位到桶位置
2. 如果桶中已有元素，遍历比较 `hashCode()` 和 `equals()`
3. 只有当 `hashCode()` 相同且 `equals()` 返回 true 时，才认为是重复元素

**所以自定义对象放入 HashSet，必须同时正确重写 `hashCode()` 和 `equals()`。**

---

## Q3：HashSet 允许 null 吗？

**A：**
允许，最多一个 null 元素。因为底层 HashMap 允许一个 null key，`put(null, PRESENT)` 只能成功一次，再次添加会覆盖而非重复。

---

## Q4：HashSet 和 HashMap 的区别？

**A：**

| 维度 | HashMap | HashSet |
|------|---------|---------|
| 结构 | 键值对 | 单列集合 |
| 底层 | HashMap | HashMap |
| key | key 是自定义，value 是值 | 元素就是 key，value 固定为 PRESENT |
| 重复判断 | key 的 hashCode + equals | 元素的 hashCode + equals |
| 容量 | 16（初始）+ 扩容 | 继承 HashMap 的容量机制 |
| null | key 和 value 都可为 null | 可以存一个 null 元素 |
| 线程安全 | ❌ 非线程安全 | ❌ 非线程安全 |

---

## Q5：HashSet 的 add 方法返回 true/false 是什么意思？

**A：**
`add(E e)` 返回 `map.put(e, PRESENT) == null`，即：
- `true`：元素**添加成功**（HashMap 中之前不存在这个 key）
- `false`：元素**添加失败**（HashMap 中已存在这个 key，被覆盖）

```java
HashSet<String> set = new HashSet<>();
System.out.println(set.add("A"));  // true
System.out.println(set.add("A"));  // false，"A" 已存在
```

---

## Q6：HashSet 的初始容量和负载因子是多少？有什么影响？

**A：**
- **初始容量**：默认 16
- **负载因子**：默认 0.75
- **扩容阈值**：capacity × loadFactor = 12

扩容机制：元素数量 > 12 时触发扩容，容量翻倍（16 → 32 → 64...）。

**调优建议**：
- 如果能预估元素数量，创建时指定初始容量：`new HashSet<>(1000)`
- 减少扩容次数，提升性能

---

## Q7：HashSet 的遍历顺序是怎样的？

**A：**
HashSet 的遍历顺序是**无序的**，不保证任何顺序：
- 不保证按照插入顺序
- 不保证按照自然顺序
- 不保证每次运行顺序一致

如果需要有序遍历，使用：
- **LinkedHashSet**：保持插入顺序
- **TreeSet**：按自然顺序/Comparator 排序

---

## Q8：HashSet 是线程安全的吗？有哪些替代方案？

**A：**
HashSet 是**非线程安全**的。多线程场景下的替代方案：

| 方案 | 说明 |
|------|------|
| `Collections.synchronizedSet(new HashSet<>())` | 同步包装，性能一般 |
| `CopyOnWriteArraySet` | 读多写少场景，读无需加锁 |
| `ConcurrentHashMap.newKeySet()` | JDK 8+，推荐使用 |

```java
// 推荐：ConcurrentHashMap 的 KeySet
Set<String> concurrentSet = ConcurrentHashMap.<String>newKeySet();
```
