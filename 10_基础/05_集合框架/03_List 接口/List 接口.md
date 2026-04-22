---
title: List 接口
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# List 接口（ArrayList / LinkedList / Vector）

## List 接口特点

`List` 是 `Collection` 的子接口，核心特点：

| 特点 | 说明 |
|------|------|
| **有序** | 元素按插入顺序排列，每个元素有索引 |
| **可重复** | 允许存储重复元素 |
| **索引访问** | 支持通过下标随机访问 `get(int index)` |

```java
public interface List<E> extends Collection<E> {
    E get(int index);            // 索引访问
    E set(int index, E element); // 替换
    void add(int index, E element); // 指定位置插入
    E remove(int index);         // 按索引删除
    int indexOf(Object o);       // 查找索引
    List<E> subList(int from, int to); // 子列表
}
```

## 三个实现类对比

| 维度 | ArrayList | LinkedList | Vector |
|------|-----------|------------|--------|
| 底层结构 | **动态数组** | **双向链表** | **动态数组** |
| 随机访问 `get(i)` | **O(1)** | O(n) | O(1) |
| 头部插入/删除 | O(n) | **O(1)** | O(n) |
| 中间插入/删除 | O(n) | O(n)（查找）+ O(1)（插入） | O(n) |
| 尾部追加 | 均摊 O(1) | O(1) | 均摊 O(1) |
| 线程安全 | ❌ 不安全 | ❌ 不安全 | ✅ 安全（`synchronized`） |
| 扩容 | 1.5 倍 | 无需扩容（节点） | 2 倍 |
| 额外实现 | — | 同时实现 `Deque` 接口 | — |
| 推荐 | ✅ 默认首选 | 频繁头尾操作 | ❌ 已过时 |

## ArrayList

```java
// 底层是 Object[] 数组
transient Object[] elementData;

// 默认初始容量 10
private static final int DEFAULT_CAPACITY = 10;

// 空数组（懒初始化）
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

## LinkedList

```java
// 同时实现 List 和 Deque
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient Node<E> first;  // 头节点
    transient Node<E> last;   // 尾节点

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
}
```

## Vector

```java
// 和 ArrayList 类似，但所有方法都加了 synchronized
public synchronized void addElement(E obj) { ... }
public synchronized E get(int index) { ... }
```

**为什么 Vector 已过时**：
- 所有方法粒度加锁，性能差
- 被 `Collections.synchronizedList()` 或并发集合（`CopyOnWriteArrayList`）替代

## 关联知识点

