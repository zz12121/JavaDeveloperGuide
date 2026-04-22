---
title: LinkedList 底层实现
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# LinkedList 底层实现（双向链表）

## 底层结构

```java
public class LinkedList<E> extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient int size = 0;
    transient Node<E> first;  // 头节点
    transient Node<E> last;   // 尾节点
}
```

- 底层是**双向链表**，每个元素封装为 `Node`
- 同时实现了 `List` 和 `Deque` 接口，**既可以当列表用，也可以当双端队列/栈用**
- 没有实现 `RandomAccess`，随机访问效率低

## Node 结构

```java
private static class Node<E> {
    E item;       // 当前元素
    Node<E> next; // 后继节点
    Node<E> prev; // 前驱节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

```
null ← [prev|item|next] ←→ [prev|item|next] ←→ [prev|item|next] → null
       first                                               last
```

## 核心操作

### 头部添加 addFirst()

```java
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;    // 空链表，first = last = newNode
    else
        f.prev = newNode;  // 原头节点的前驱指向新节点
    size++;
    modCount++;
}
```

### 尾部添加 addLast()

```java
public void addLast(E e) {
    linkLast(e);
}

private void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

### 删除节点

```java
// 删除头节点
public E removeFirst() {
    final Node<E> f = first;
    if (f == null) throw new NoSuchElementException();
    return unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;  // help GC
    f.next = null;
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

### 随机访问 get()

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    // 根据索引离头还是离尾更近来决定遍历方向
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

> 折半查找优化：如果索引在前半部分从头遍历，后半部分从尾遍历。但时间复杂度仍然是 **O(n)**。

## 性能特点

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 头部插入/删除 | **O(1)** | 直接操作 first 指针 |
| 尾部插入/删除 | **O(1)** | 直接操作 last 指针 |
| 中间插入/删除 | O(n) | 需要先遍历找到位置 |
| 随机访问 get(i) | O(n) | 需要遍历链表 |
| 查找 contains | O(n) | 遍历查找 |

## LinkedList 作为栈和队列

```java
// 作为栈（后进先出）
LinkedList<String> stack = new LinkedList<>();
stack.push("a");    // addFirst
stack.push("b");
stack.pop();        // removeFirst → "b"

// 作为队列（先进先出）
LinkedList<String> queue = new LinkedList<>();
queue.offer("a");   // addLast
queue.offer("b");
queue.poll();       // removeFirst → "a"
```

## 关联知识点

