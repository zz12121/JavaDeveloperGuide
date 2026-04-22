---
title: ArrayList 底层实现
tags:
  - Java/集合框架
  - 源码型
module: 05_集合框架
created: 2026-04-18
---

# ArrayList 底层实现（扩容机制）

## 底层结构

```java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    transient Object[] elementData;  // 存储元素的数组
    private int size;                // 当前元素个数
}
```

- 底层是 **`Object[]` 数组**
- 默认初始容量为 **10**（JDK 7 及之前直接创建；JDK 8 懒初始化，首次 add 时才创建）
- 实现了 `RandomAccess` 接口，支持快速随机访问

## 扩容机制（核心）

### 关键属性

```java
private static final int DEFAULT_CAPACITY = 10;        // 默认容量
private static final Object[] EMPTY_ELEMENTDATA = {};   // 指定容量为 0 时的空数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; // 无参构造的空数组
```

### add() 流程

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 1. 检查是否需要扩容
    elementData[size++] = e;            // 2. 赋值
    return true;
}
```

### 扩容三步曲

```
add(e)
  → ensureCapacityInternal(minCapacity)
    → calculateCapacity(elementData, minCapacity)   // 计算所需最小容量
      → 如果是空数组 → 返回 max(10, minCapacity)
      → 否则 → 返回 minCapacity
    → ensureExplicitCapacity(minCapacity)            // 确认是否真的需要扩容
      → modCount++
      → if (minCapacity > elementData.length) → grow(minCapacity)
    → grow(minCapacity)                              // 真正的扩容
      → int newCapacity = oldCapacity + (oldCapacity >> 1);  // newCapacity = 1.5 * oldCapacity
      → if (newCapacity < minCapacity) → newCapacity = minCapacity
      → if (newCapacity > MAX_ARRAY_SIZE) → hugeCapacity(minCapacity)
      → elementData = Arrays.copyOf(elementData, newCapacity);  // 数组拷贝
```

### 扩容要点

| 要点 | 说明 |
|------|------|
| 扩容倍数 | `newCapacity = oldCapacity + (oldCapacity >> 1)` = **1.5 倍** |
| 触发时机 | `minCapacity > elementData.length` 时 |
| 底层操作 | `Arrays.copyOf()` → 底层调用 `System.arraycopy()`（**native 方法**） |
| 空间代价 | 每次扩容丢弃旧数组，新数组，浪费约 33% 空间 |
| 时间代价 | 扩容时需要 O(n) 复制所有元素 |

## 预估容量的好处

```java
// 知道大概数据量时，提前指定容量，避免多次扩容
List<String> list = new ArrayList<>(1000);
```

## 序列化

`elementData` 被 `transient` 修饰，不会被默认序列化：

```java
transient Object[] elementData;
```

ArrayList 自定义了 `writeObject` / `readObject`，只序列化实际有值的元素（`0 ~ size-1`），避免浪费空间。

## 关联知识点
