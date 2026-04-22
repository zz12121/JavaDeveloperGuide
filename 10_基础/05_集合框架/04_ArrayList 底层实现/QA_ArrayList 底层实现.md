---
title: ArrayList 底层实现面试题
tags:
  - Java/集合框架
  - 源码型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# ArrayList 底层实现

## Q1：ArrayList 的底层实现原理？

**A：**
ArrayList 底层是 **`Object[]` 动态数组**。默认初始容量为 10（JDK 8 懒初始化，首次 `add` 时才创建数组）。当数组空间不足时自动扩容为原来的 **1.5 倍**。

---

## Q2：ArrayList 扩容机制是怎样的？

**A：**
1. `add(e)` 时先调用 `ensureCapacityInternal(size + 1)` 检查容量
2. 如果当前数组容量不够，调用 `grow()` 扩容
3. 新容量 = 旧容量 + 旧容量右移 1 位 = **1.5 倍**
4. 如果新容量仍不够，直接用所需容量
5. 通过 `Arrays.copyOf()` 将旧数组元素复制到新数组

```java
int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
elementData = Arrays.copyOf(elementData, newCapacity);
```

---

## Q3：ArrayList 为什么用 transient 修饰 elementData？

**A：**
因为数组可能只填了一部分，直接序列化整个数组会浪费空间。ArrayList 自定义了 `writeObject()`，只序列化 `0 ~ size-1` 的有效元素，避免序列化多余空间。

---

## Q4：ArrayList 是如何实现随机访问的？

**A：**
ArrayList 实现了 `RandomAccess` 接口（标记接口），底层是数组，`get(index)` 直接通过下标访问，时间复杂度 O(1)。

```java
public E get(int index) {
    rangeCheck(index);
    return elementData(index);  // 直接数组下标访问
}
```
