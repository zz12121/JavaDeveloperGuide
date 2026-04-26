---
title: CopyOnWriteArrayList
tags:
  - Java/并发
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# CopyOnWriteArrayList

## 核心结论

CopyOnWriteArrayList（写时复制）：**读不加锁，写时复制新数组**。通过 volatile 引用切换实现读写分离，适合**读多写少**场景。

## 深度解析

### 原理

```
CopyOnWriteArrayList
├── array: volatile Object[]（volatile 引用）

读操作：直接读 array，无锁
写操作：
  1. 加锁（ReentrantLock）
  2. 复制新数组（Arrays.copyOf）
  3. 在新数组上修改
  4. 将 array 引用指向新数组（volatile 写）
  5. 释放锁
```

### add 源码

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements); // volatile 写
        return true;
    } finally {
        lock.unlock();
    }
}

final Object[] getArray()  { return array; }
final void setArray(Object[] a) { array = a; }
```

### get 源码（无锁）

```java
public E get(int index) {
    return get(getArray(), index); // volatile 读 array
}
```

### 迭代器特性

- 迭代器遍历的是创建时的**数组快照**
- 迭代过程中 list 的修改不影响当前迭代
- 迭代器**不支持 remove/add**（抛 UnsupportedOperationException）

### 优缺点

**优点**：读无锁、不抛 ConcurrentModificationException、迭代器安全

**缺点**：每次写复制整个数组（O(n)）、写加锁单线程写、内存翻倍、弱一致性

### 适用场景

- 读多写少（如黑白名单、事件监听器列表、配置列表）
- 不适合大数组 + 频繁写


# CopyOnWriteArrayList迭代器弱一致性

## 核心结论

CopyOnWriteArrayList 迭代器遍历的是**创建时的快照数组**，不反映后续修改，不支持 remove/add/set 操作。这是弱一致性的典型体现。

## 深度解析

### COW 机制回顾

```
CopyOnWriteArrayList
├── array: volatile Object[]（核心数据结构）

读操作：直接读 array，无锁，O(1)
写操作：
  1. ReentrantLock 加锁
  2. 复制新数组（Arrays.copyOf）
  3. 在新数组上修改
  4. volatile 写切换引用
  5. 释放锁
```

**关键点**：volatile 保证写线程的修改对读线程可见，但读操作不加锁。

### 迭代器实现原理

```java
// 获取迭代器
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}

// COWIterator 内部
private static class COWIterator<E> implements ListIterator<E> {
    private final Object[] snapshot;  // 快照数组
    private int cursor;                // 游标
    
    COWIterator(Object[] elements, int initialCursor) {
        snapshot = elements;           // 直接引用当前 array
        cursor = initialCursor;
    }
    
    // next/hasNext 基于快照数组操作
    public E next() {
        return (E) snapshot[cursor++];
    }
}
```

**核心**：`iterator()` 调用时直接获取 `getArray()` 返回的引用作为快照，之后 list 的修改（通过 volatile 切换 array）不会影响 snapshot。

### 弱一致性的体现

|操作|结果|
|---|---|
|迭代器创建后，线程 A 修改 list|迭代器**不可见**新元素|
|迭代器创建后，其他线程遍历 list|可能看到**不同数据**|
|迭代过程中，线程 B 删除元素|迭代器可能仍返回已删除元素|

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");
list.add("C");

Iterator<String> iter = list.iterator();  // 快照：[A, B, C]

list.add("D");  // array 引用切换到新数组

// 迭代器仍遍历旧快照
iter.forEachRemaining(System.out::println);  // 输出：A B C（不含 D）
```

### 不支持修改操作

迭代器**不支持**以下操作，调用抛 `UnsupportedOperationException`：

- `Iterator.remove()`
- `ListIterator.set(E e)`
- `ListIterator.add(E e)`

```java
Iterator<String> iter = list.iterator();
iter.remove();  // UnsupportedOperationException
iter.set("X");  // UnsupportedOperationException
iter.add("Y");  // UnsupportedOperationException
```

**原因**：快照是只读的，且修改快照无意义（无法影响原 list）。

### 与 ConcurrentHashMap.KeySetView 迭代器对比

|特性|CopyOnWriteArrayList 迭代器|ConcurrentHashMap.KeySetView 迭代器|
|---|---|---|
|数据来源|快照数组|桶节点的 next 链表|
|一致性|弱一致（快照）|弱一致（无锁遍历）|
|remove()|不支持|不支持|
|并发修改|安全（快照隔离）|安全（单桶无锁）|
|可见性|不反映后续修改|可能跳元素|

两者都是弱一致性，但机制不同：COW 用快照，CHM 用无锁链表遍历。

### 适用场景

|场景|适用性|原因|
|---|---|---|
|读多写少，容忍短暂不一致|✅|迭代期间数据可能不是最新|
|配置/规则列表遍历|✅|快照保证迭代安全|
|频繁写的列表遍历|❌|迭代器看不到新数据|
|需要在遍历时删除元素|❌|不支持 remove()|
|强一致性要求|❌|弱一致性设计|

### 代码示例

```java
public class COWIteratorDemo {
    public static void main(String[] args) {
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>();
        list.add(1);
        list.add(2);
        list.add(3);
        
        // 获取迭代器（快照）
        Iterator<Integer> iter = list.iterator();
        
        // 主线程添加元素
        list.add(4);
        list.remove(0);
        
        // 迭代器仍看到旧数据
        System.out.println("Iterator sees: " + iter.next());  // 1
        
        // 新迭代器看到新数据
        System.out.println("New iterator sees:");
        list.iterator().forEachRemaining(System.out::println);  // 2 3 4
    }
}
```

## 关联知识点
