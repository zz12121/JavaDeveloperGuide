---
title: TreeSet 底层实现
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# TreeSet 底层实现（红黑树）

## 底层原理

TreeSet 底层是 **TreeMap**（红黑树），元素作为 TreeMap 的 key：

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {

    private transient NavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public TreeSet() {
        this.m = new TreeMap<>();  // 自然排序
    }

    public TreeSet(Comparator<? super E> comparator) {
        this.m = new TreeMap<>(comparator);  // 自定义排序
    }
}
```

## 红黑树特点

| 特点 | 说明 |
|------|------|
| 自平衡二叉搜索树 | 插入/删除后自动调整，保持平衡 |
| 查找/插入/删除 | O(log n) |
| 有序 | 中序遍历就是排序结果 |
| 节点着色 | 每个节点为红或黑，通过规则保持平衡 |

## 排序方式

### 自然排序（Comparable）

元素必须实现 `Comparable` 接口：

```java
Set<Integer> set = new TreeSet<>();
set.add(30); set.add(10); set.add(20);
// 遍历: 10, 20, 30（自然升序）

// 自定义对象
class Student implements Comparable<Student> {
    String name;
    int score;

    @Override
    public int compareTo(Student o) {
        return this.score - o.score;  // 按分数排序
    }
}
```

### 自定义排序（Comparator）

```java
Set<String> set = new TreeSet<>(Comparator.reverseOrder());
set.add("c"); set.add("a"); set.add("b");
// 遍历: c, b, a（降序）

// 按字符串长度排序
Set<String> set2 = new TreeSet<>(Comparator.comparingInt(String::length));
```

## 去重原理

TreeSet 的去重依赖 `compareTo()` 或 `Comparator.compare()`：

```java
public boolean add(E e) {
    return m.put(e, PRESENT) == null;
}

// TreeMap.put() 中：
// 如果 compareTo() 返回 0 → 认为是同一个 key → 不添加（去重）
// 如果 compareTo() 返回非 0 → 放入正确位置
```

> **注意**：TreeSet 去重只用 `compareTo()`，**不用 `equals()`**！这可能导致 TreeSet 和 HashSet 的去重结果不一致。

## NavigableSet 能力

TreeSet 实现了 `NavigableSet`，提供范围查询：

```java
TreeSet<Integer> set = new TreeSet<>();
set.addAll(Arrays.asList(1, 3, 5, 7, 9));

set.first();                    // 1（最小）
set.last();                     // 9（最大）
set.lower(5);                   // 3（严格小于）
set.floor(5);                   // 5（小于等于）
set.higher(5);                  // 7（严格大于）
set.ceiling(5);                 // 5（大于等于）
set.subSet(3, 8);               // [3, 5, 7]（子集）
set.headSet(5);                 // [1, 3]（前缀）
set.tailSet(5);                 // [5, 7, 9]（后缀）
```

## 注意事项

1. **不允许 null**：`compareTo(null)` 会抛 NPE
2. **compareTo 和 equals 不一致**可能导致 HashSet 和 TreeSet 行为不同
3. **性能**：O(log n)，比 HashSet 的 O(1) 慢，但提供了排序

## 联系

- [[52_Set接口（HashSet-LinkedHashSet-TreeSet）]]
- [[56_Map接口（HashMap-LinkedHashMap-TreeMap-Hashtable-ConcurrentHashMap）]]
