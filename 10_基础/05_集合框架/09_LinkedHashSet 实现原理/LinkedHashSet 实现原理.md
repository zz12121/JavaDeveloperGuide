---
title: LinkedHashSet 实现原理
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# LinkedHashSet 实现原理（维护插入顺序）

## 底层原理

LinkedHashSet 继承 HashSet，内部使用 **LinkedHashMap** 替代 HashMap：

```java
public class LinkedHashSet<E> extends HashSet<E> {
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);  // dummy 参数
    }
}
```

> 注意：`LinkedHashSet` 调用了一个 `HashSet` 的**包级私有构造器**，该构造器的第三个参数 `dummy` 为 true 时使用 LinkedHashMap。

## LinkedHashMap 的双向链表

LinkedHashMap 在 HashMap 基础上增加了一个**双向链表**，维护所有节点的插入顺序（或访问顺序）：

```java
public class LinkedHashMap<K, V> extends HashMap<K, V> {
    transient LinkedHashMap.Entry<K, V> head;  // 双向链表头
    transient LinkedHashMap.Entry<K, V> tail;  // 双向链表尾

    static class Entry<K, V> extends HashMap.Node<K, V> {
        Entry<K, V> before, after;  // 双向链表指针
        Entry(int hash, K key, V value, Node<K, V> next) {
            super(hash, key, value, next);
        }
    }
}
```

```
HashMap 桶结构（快速查找）：     双向链表（维护顺序）：

桶0 → [Node]                      null ← [A] ←→ [B] ←→ [C] → null
桶1 → [Node] → [Node]                    ↑              ↑
桶2 → null                              head           tail

遍历顺序：A → B → C（插入顺序）
```

## 特点

| 特点 | 说明 |
|------|------|
| 去重 | 和 HashSet 一样，基于 `hashCode()` + `equals()` |
| 顺序 | **保持插入顺序**（遍历时按添加顺序输出） |
| 性能 | 插入/查找/删除 O(1)，比 HashSet 略慢（维护链表的开销） |
| 内存 | 比 HashSet 多一个双向链表，内存占用略大 |

## 使用示例

```java
Set<String> set = new LinkedHashSet<>();
set.add("banana");
set.add("apple");
set.add("cherry");

// 遍历顺序 = 插入顺序
for (String s : set) {
    System.out.println(s);  // banana, apple, cherry
}
```

## 适用场景

- 需要**去重 + 保持插入顺序**时使用
- 典型场景：URL 去重爬虫（保持访问顺序）、菜单项去重等

## 关联知识点
