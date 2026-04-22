---
title: HashSet 底层实现
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# HashSet 底层实现（HashMap）

## 底层原理

HashSet 的底层就是 **HashMap**，每个 HashSet 元素作为 HashMap 的 key，value 是一个固定的空对象。

```java
public class HashSet<E> extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();  // 固定 value
}
```

## 核心方法源码

### add()

```java
public boolean add(E e) {
    return map.put(e, PRESENT) == null;
    // put 返回 null → 之前没有这个 key → 添加成功 → return true
    // put 返回旧 value → 之前已有这个 key → 添加失败 → return false
}
```

### remove()

```java
public boolean remove(Object o) {
    return map.remove(o) == PRESENT;
}
```

### contains()

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

### size()

```java
public int size() {
    return map.size();
}
```

## 去重原理

HashSet 的去重依赖 HashMap 的 key 唯一性：

```
add(e)
  → map.put(e, PRESENT)
    → hash(e) 定位桶
    → 桶为空 → 直接放入（新元素）
    → 桶不为空 → 遍历链表/红黑树
      → e.hashCode() 相同 && e.equals(other) → 已存在，不重复
      → 否则 → 添加到链表/红黑树尾部（新元素）
```

关键：**先比较 `hashCode()`，再比较 `equals()`**

## 重要结论

1. HashSet **完全依赖** HashMap，所有操作都是对 HashMap 的一层包装
2. 去重 = HashMap key 不重复
3. HashSet 的性能 = HashMap 的性能（O(1) 查找/插入/删除）
4. HashSet 允许存储一个 null 元素（HashMap 允许一个 null key）

## 注意事项

如果自定义对象放入 HashSet，必须重写 `hashCode()` 和 `equals()`，否则两个"逻辑相同"的对象会被当作不同对象：

```java
// 不重写：两个内容相同但不同引用的对象都会被添加
// 重写后：内容相同的对象只会保留一个
class User {
    String name;
    int age;

    @Override
    public boolean equals(Object o) { ... }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

## 关联知识点
