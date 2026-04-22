---
title: Deque 接口
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# Deque 接口（双端队列）

## Deque 接口特点

`Deque`（Double-Ended Queue）继承自 `Queue`，支持在**两端**添加和删除元素：

```java
public interface Deque<E> extends Queue<E> {
    // 头部操作
    void addFirst(E e);       // 头部添加（失败抛异常）
    void addLast(E e);        // 尾部添加（失败抛异常）
    boolean offerFirst(E e);  // 头部添加（推荐）
    boolean offerLast(E e);   // 尾部添加（推荐）

    E removeFirst();          // 头部移除（失败抛异常）
    E removeLast();           // 尾部移除（失败抛异常）
    E pollFirst();            // 头部移除（推荐）
    E pollLast();             // 尾部移除（推荐）

    E getFirst();             // 获取头部（失败抛异常）
    E getLast();              // 获取尾部（失败抛异常）
    E peekFirst();            // 获取头部（推荐）
    E peekLast();             // 获取尾部（推荐）

    // 栈操作
    void push(E e);           // 等价于 addFirst
    E pop();                  // 等价于 removeFirst
}
```

## Deque = Queue + Stack

Deque 同时具备**队列**和**栈**的功能：

```
队列（FIFO）：offerLast / pollFirst
栈（LIFO）： push=offerFirst / pop=removeFirst
```

## 主要实现类

| 实现类 | 底层 | 特点 |
|--------|------|------|
| LinkedList | 双向链表 | 同时实现 List 和 Deque |
| ArrayDeque | **循环数组** | 高性能，推荐替代 Stack |

### ArrayDeque

```java
// 循环数组实现，比 LinkedList 更快
public class ArrayDeque<E> extends AbstractCollection<E>
    implements Deque<E>, Cloneable, Serializable {

    transient Object[] elements;  // 底层数组
    int head;                     // 头指针
    int tail;                     // 尾指针
}
```

**循环数组原理**：

```
elements[] = [e3, e4, _, _, _, e1, e2]
              ↑tail         ↑head

头部添加：head 左移
尾部添加：tail 右移
到达边界时环绕
```

### ArrayDeque vs LinkedList vs Stack

| 维度 | ArrayDeque | LinkedList | Stack |
|------|-----------|------------|-------|
| 底层 | 循环数组 | 双向链表 | 数组（Vector） |
| 线程安全 | ❌ | ❌ | ✅（过时） |
| 作为栈性能 | **最快** | 较慢 | 慢 |
| 作为队列性能 | **最快** | 较慢 | 不适合 |
| 推荐 | ✅ 替代 Stack | 特殊场景 | ❌ 已过时 |

## 使用示例

### 作为栈（替代 Stack）

```java
// ✅ 推荐：ArrayDeque 作为栈
Deque<String> stack = new ArrayDeque<>();
stack.push("a");   // offerFirst
stack.push("b");   // offerFirst
stack.pop();       // removeFirst → "b"
stack.peek();      // peekFirst → "a"
```

### 作为队列

```java
Deque<String> queue = new ArrayDeque<>();
queue.offerLast("a");   // 尾部入队
queue.offerLast("b");
queue.pollFirst();      // 头部出队 → "a"
```

### 作为双端队列

```java
Deque<String> deque = new ArrayDeque<>();
deque.offerFirst("middle");  // [middle]
deque.offerFirst("left");    // [left, middle]
deque.offerLast("right");    // [left, middle, right]

deque.pollFirst();  // "left"
deque.pollLast();   // "right"
```

## 为什么 ArrayDeque 比 LinkedList 快

1. **内存连续**：数组在内存中连续，CPU 缓存命中率高
2. **无对象创建**：LinkedList 每次 add 需要 `new Node()`，ArrayDeque 只需数组赋值
3. **无指针开销**：LinkedList 每个节点 2 个指针，ArrayDeque 只需 2 个 int 索引

## 关联知识点

