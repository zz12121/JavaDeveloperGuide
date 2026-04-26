---
title: CopyOnWriteArrayList
tags:
  - Java/并发
  - 问答
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# CopyOnWriteArrayList

## Q1：CopyOnWriteArrayList 的读写机制是什么？

**A**：

- **读**：直接 volatile 读 array 引用，无锁，O(1)
- **写**：ReentrantLock 加锁 → 复制新数组 → 修改新数组 → volatile 写切换引用 → 释放锁

读操作完全无锁，写操作需要复制整个数组并加锁。

---

## Q2：CopyOnWriteArrayList 有什么缺点？

**A**：

1. **写性能差**：每次写都复制整个数组，O(n) 时间和内存
2. **写并发度低**：用 ReentrantLock，写操作串行
3. **内存占用**：写时新旧数组共存，内存翻倍
4. **弱一致性**：读到的可能不是最新写入的值
5. **大数组不适用**：数组越大，复制开销越大

---

## Q3：CopyOnWriteArrayList 的迭代器为什么不会抛 ConcurrentModificationException？

**A**：因为迭代器持有的是**创建时的数组快照引用**，而不是原始 list 的引用。

```java
// 迭代器创建时保存当前数组
COWIterator(Object[] elements) {
    snapshot = elements; // 快照
}
```

后续对 list 的 add/remove 不会影响快照数组，所以迭代安全。

但这也意味着迭代器**看不到迭代过程中新增的元素**（弱一致性），且**不支持 remove/add**。

---

## Q4：CopyOnWriteArrayList 适合什么场景？

**A**：**读远多于写**的场景：

1. 事件监听器列表（大量遍历，很少添加/移除）
2. 黑白名单
3. 配置列表
4. 缓存快照

不适合：频繁写、大数组、强一致性要求。


# CopyOnWriteArrayList迭代器弱一致性

## Q1：CopyOnWriteArrayList 的迭代器为什么是弱一致性的？

**A**：

弱一致性源于迭代器的**快照机制**：

1. **快照创建**：调用 `iterator()` 时，直接获取当前 `array` 引用作为快照
2. **隔离性**：后续 list 的写操作会创建新数组并切换 `array` 引用，但迭代器持有的快照引用不变
3. **不可见新数据**：迭代器永远看不到创建之后新增的元素

```java
// 核心源码
COWIterator(Object[] elements) {
    this.snapshot = elements;  // 创建时的数组引用
}
```

**实际表现**：

- 迭代器创建后，其他线程 add/remove 元素 → 迭代器不可见
- 迭代器看不到"最新"数据，只能看到创建那一刻的数据状态

弱一致性 = **时间维度上的一致性削弱**：牺牲实时性，换取无锁读的安全性。

---

## Q2：CopyOnWriteArrayList 迭代器为什么不支持 remove()？

**A**：

不支持 `remove()/set()/add()` 是快照机制的**必然结果**：

|设计选择|原因|
|---|---|
|**快照只读**|迭代器持有的是原数组引用，直接修改会影响原 list（违背 COW 语义）|
|**修改快照无效**|即使修改了快照，原 list 的 array 引用未变，修改不会反映到 list|
|**避免复杂度**|COW 的核心是"读写分离"，让迭代器能无锁遍历|

```java
// 调用 remove 实际会抛异常
public void remove() {
    throw new UnsupportedOperationException();
}
```

**对比**：普通 ArrayList 迭代器的 remove() 可以修改底层数组，因为迭代器和 list 共享同一数组引用。

**替代方案**：如果需要在遍历时删除，可用普通 for 循环 + list.remove()（会触发 COW，效率低）。

---

## Q3：弱一致性在实际中会有什么问题？

**A**：

弱一致性可能带来以下问题：

### 1. 数据滞后

```
线程 A（迭代器）：看到 [1, 2, 3]
线程 B：add(4), remove(1)  → list 现在是 [2, 3, 4]
线程 A 迭代结果：[1, 2, 3]（缺少 4，看到已删除的 1）
```

### 2. 业务逻辑错误

- **缓存场景**：迭代器遍历后，发现数据"不存在"（实际已被删除）
- **去重逻辑**：迭代器可能对"已删除"的元素重复处理
- **计数统计**：迭代期间的统计数据与最终数据不一致

### 3. 解决方案

|方案|适用场景|代价|
|---|---|---|
|每次遍历前获取新迭代器|强一致性要求|重复创建迭代器开销|
|使用普通 List + 同步|需要实时数据|失去并发读优势|
|接受短暂不一致|读多写少，允许脏读|业务需容忍|
|改用 CopyOnWriteArrayList + copyTo|遍历前快照|额外的复制开销|

### 4. 典型踩坑案例

```java
// ❌ 错误：期望遍历到所有元素
CopyOnWriteArrayList<Request> requests = ...;
for (Request r : requests) {
    if (r.isProcessed()) {
        requests.remove(r);  // UnsupportedOperationException
    }
}

// ✅ 正确：收集后统一处理
List<Request> toRemove = new ArrayList<>();
for (Request r : requests) {
    if (r.isProcessed()) {
        toRemove.add(r);
    }
}
requests.removeAll(toRemove);
```