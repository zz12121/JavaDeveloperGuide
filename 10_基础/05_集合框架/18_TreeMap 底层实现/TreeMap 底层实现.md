---
title: TreeMap 底层实现
tags:
  - Java/集合框架
  - 源码型
module: 05_集合框架
created: 2026-04-25
---

# TreeMap 底层实现（红黑树）

## 底层结构

TreeMap 底层是**红黑树（Red-Black Tree）**，一种自平衡的二叉搜索树：

```java
public class TreeMap<K, V> extends AbstractMap<K, V>
    implements NavigableMap<K, V>, Cloneable, java.io.Serializable {

    private final Comparator<? super K> comparator;  // 比较器
    private transient Entry<K, V> root;              // 根节点
    private transient int size = 0;
}
```

**红黑树节点结构：**

```java
static class Entry<K, V> implements Map.Entry<K, V> {
    K key;
    V value;
    Entry<K, V> left;    // 左子节点
    Entry<K, V> right;   // 右子节点
    Entry<K, V> parent;  // 父节点
    boolean color = BLACK;  // 节点颜色（红或黑）
}
```

---

## 红黑树五大特性

| 特性 | 说明 |
|------|------|
| ① 每个节点非红即黑 | 颜色属性只有两种 |
| ② 根节点是黑色 |  |
| ③ 叶子节点（NIL）是黑色 | 红黑树的叶子节点是空节点 |
| ④ 红节点的子节点都是黑色 | 不能出现两个相邻的红节点 |
| ⑤ 每个节点到叶子节点的路径上黑色节点数相同 | **黑高平衡** |

这些特性保证了红黑树的**近似平衡**，查找/插入/删除都是 **O(log n)**。

---

## 排序方式

### 自然排序（Comparable）

元素必须实现 `Comparable` 接口：

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(30); set.add(10); set.add(20);
// 遍历: 10, 20, 30

// 自定义对象实现 Comparable
class Student implements Comparable<Student> {
    String name;
    int score;

    @Override
    public int compareTo(Student o) {
        return this.score - o.score;  // 按分数升序
    }
}
```

### 自定义排序（Comparator）

```java
// 降序排列
TreeSet<Integer> set = new TreeSet<>(Comparator.reverseOrder());
set.add(1); set.add(3); set.add(2);
// 遍历: 3, 2, 1

// 按字符串长度排序
TreeSet<String> set2 = new TreeSet<>(Comparator.comparingInt(String::length));
```

---

## put 流程

```
put(key, value)
│
├── 1. 根节点为空？
│   ├── 是 → 直接创建根节点（黑色）→ 结束
│   └── 否 → 继续
│
├── 2. 比较 key（用 comparator 或 compareTo）
│   ├── key < node.key → 往左子树走
│   ├── key > node.key → 往右子树走
│   └── key == node.key → 替换 value → 结束
│
├── 3. 找到插入位置，创建新节点（红色）
│
├── 4. 调整红黑树（fixAfterInsertion）
│   ├── 新节点是根节点 → 染成黑色
│   ├── 新节点不是根节点 → 检查并修复红黑树特性
│
└── 结束
```

### 核心源码

```java
public V put(K key, V value) {
    Entry<K, V> t = root;
    if (t == null) {
        // 根节点特殊处理
        root = new Entry<>(key, value, null);
        size = 1;
        return null;
    }

    int cmp;
    Entry<K, V> parent;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 自定义比较器
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);  // key 相同，替换 value
        } while (t != null);
    } else {
        // 自然排序
        Objects.requireNonNull(key);
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);
        } while (t != null);
    }

    // 创建新节点
    Entry<K, V> e = new Entry<>(key, value, parent);
    if (cmp < 0) parent.left = e;
    else parent.right = e;

    fixAfterInsertion(e);  // 调整红黑树
    size++;
    return null;
}
```

---

## 红黑树自平衡调整

### 两种基本操作：左旋和右旋

```java
// 左旋：以 p 为支点，其右子节点变为父节点
private void rotateLeft(Entry<K, V> p) {
    if (p != null) {
        Entry<K, V> r = p.right;
        p.right = r.left;
        if (r.left != null) r.left.parent = p;
        r.parent = p.parent;
        if (p.parent == null) root = r;
        else if (p.parent.left == p) p.parent.left = r;
        else p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}

// 右旋：以 p 为支点，其左子节点变为父节点
private void rotateRight(Entry<K, V> p) {
    if (p != null) {
        Entry<K, V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null) root = l;
        else if (p.parent.right == p) p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

### 插入修复（fixAfterInsertion）

新节点默认红色，插入后需要修复的情况：

| 情况 | 条件 | 动作 |
|------|------|------|
| 情况1 | 新节点是根节点 | 直接染黑 |
| 情况2 | 新节点的父节点是黑色 | 无需调整 |
| 情况3 | 新节点的父节点是红色 + 叔节点也是红色 | 父、叔染黑，祖父染红，递归向上处理 |
| 情况4 | 新节点的父节点是红色 + 叔节点是黑色 + 新节点是右子 | 先左旋变成情况5 |
| 情况5 | 新节点的父节点是红色 + 叔节点是黑色 + 新节点是左子 | 父染黑、祖父染红、右旋祖父 |

---

## get 流程

```java
public V get(Object key) {
    Entry<K, V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp < 0) p = p.left;
        else if (cmp > 0) p = p.right;
        else return p.value;
    }
    return null;
}
```

查找就是标准的二叉搜索树查找，时间复杂度 **O(log n)**。

---

## NavigableMap 能力

TreeMap 实现了 `NavigableMap`，提供丰富的导航操作：

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "A"); map.put(3, "C"); map.put(5, "E"); map.put(7, "G");

// 导航方法
map.firstKey();                    // 1（最小 key）
map.lastKey();                     // 7（最大 key）
map.lowerKey(5);                   // 3（严格小于 5）
map.floorKey(5);                   // 5（小于等于 5）
map.higherKey(5);                  // 7（严格大于 5）
map.ceilingKey(5);                 // 5（大于等于 5）

// 范围操作
map.subMap(1, 5);                  // {1=A, 3=C}（前闭后开）
map.subMap(1, true, 5, true);      // {1=A, 3=C, 5=E}（前开后闭）
map.headMap(5);                    // {1=A, 3=C}（< 5）
map.tailMap(3);                    // {3=C, 5=E, 7=G}（>= 3）

// 获取第一个/最后一个 entry
map.firstEntry();                   // Map.Entry(1, "A")
map.lastEntry();                   // Map.Entry(7, "G")
map.lowerEntry(5);                 // Map.Entry(3, "C")
map.higherEntry(5);                // Map.Entry(7, "G")
```

---

## 性能特点

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| put | O(log n) | 查找位置 + 最多两次旋转 |
| get | O(log n) | 二叉搜索树查找 |
| remove | O(log n) | 查找 + 删除 + 调整 |
| 遍历 | O(n) | 中序遍历 |

---

## HashMap vs TreeMap vs LinkedHashMap

| 维度 | HashMap | LinkedHashMap | TreeMap |
|------|---------|---------------|---------|
| 底层结构 | 数组+链表+红黑树 | HashMap+双向链表 | 红黑树 |
| 遍历顺序 | **无序** | **插入顺序/访问顺序** | **Key 排序** |
| 性能 | O(1) 均摊 | O(1) 均摊 | O(log n) |
| null key | ✅ 允许 | ✅ 允许 | ❌ 不允许 |
| null value | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| 适用场景 | 普通映射 | 有序遍历、LRU | 排序需求 |

---

## 注意事项

1. **Key 不能为 null**：`compareTo(null)` 会抛 NPE
2. **compareTo 和 equals 应一致**：TreeMap 用 `compareTo` 判断相等，和 HashMap 的 `equals` 可能不一致
3. **线程不安全**：多线程应使用 `Collections.synchronizedSortedMap()` 或 `ConcurrentSkipListMap`

---

## 关联知识点

- [[HashMap 底层实现]]
- [[LinkedHashMap 底层实现]]
- [[Set 接口]]
- [[TreeSet 底层实现]]
