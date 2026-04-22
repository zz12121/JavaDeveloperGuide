---
title: Set 接口
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# Set 接口（HashSet / LinkedHashSet / TreeSet）

## Set 接口特点

`Set` 继承自 `Collection`，核心特点：

| 特点 | 说明 |
|------|------|
| **不重复** | 不允许存储重复元素（由 `equals()` 和 `hashCode()` 判断） |
| **最多一个 null** | HashSet 和 LinkedHashSet 允许一个 null；TreeSet 不允许 |
| **无索引** | 没有 `get(index)` 方法，不支持按位置访问 |

## 三个实现类对比

| 维度 | HashSet | LinkedHashSet | TreeSet |
|------|---------|---------------|---------|
| 底层实现 | **HashMap** | **LinkedHashMap** | **NavigableMap（TreeMap）** |
| 元素顺序 | **无序** | **插入顺序** | **排序（自然序 / Comparator）** |
| null 值 | ✅ 允许 1 个 | ✅ 允许 1 个 | ❌ 不允许 |
| 查找/插入/删除 | O(1) | O(1) | O(log n) |
| 去重依据 | `hashCode()` + `equals()` | `hashCode()` + `equals()` | `compareTo()` 或 `Comparator` |
| 线程安全 | ❌ | ❌ | ❌ |

## HashSet

```java
// 底层就是 HashMap，value 是固定的 PRESENT 对象
private transient HashMap<E, Object> map;
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT) == null;  // key 是元素，value 固定
}
```

## LinkedHashSet

```java
// 继承 HashSet，内部用 LinkedHashMap
public class LinkedHashSet<E> extends HashSet<E> {
    public LinkedHashSet() {
        super(new LinkedHashMap<>());  // 调用 HashSet 的包级构造器
    }
}
```

- 在 HashMap 基础上维护了一个双向链表，记录插入顺序
- 遍历时按插入顺序输出

## TreeSet

```java
// 底层是 TreeMap（红黑树）
private transient NavigableMap<E, Object> m;

public TreeSet() {
    this.m = new TreeMap<>();  // 使用自然排序
}

public TreeSet(Comparator<? super E> comparator) {
    this.m = new TreeMap<>(comparator);  // 自定义排序
}
```

- 元素自动排序
- 不允许 null（compareTo(null) 会抛 NPE）

## 使用示例

```java
// HashSet：无序，去重
Set<String> hashSet = new HashSet<>();
hashSet.add("c"); hashSet.add("a"); hashSet.add("b");
// 遍历顺序不确定: [a, b, c] 或 [c, a, b] ...

// LinkedHashSet：保持插入顺序
Set<String> linkedSet = new LinkedHashSet<>();
linkedSet.add("c"); linkedSet.add("a"); linkedSet.add("b");
// 遍历顺序: [c, a, b]

// TreeSet：自然排序
Set<String> treeSet = new TreeSet<>();
treeSet.add("c"); treeSet.add("a"); treeSet.add("b");
// 遍历顺序: [a, b, c]
```

## 选择指南

| 需求 | 选择 |
|------|------|
| 只需要去重，不关心顺序 | **HashSet**（最快） |
| 需要去重 + 保持插入顺序 | **LinkedHashSet** |
| 需要去重 + 排序 | **TreeSet** |

## 关联知识点

