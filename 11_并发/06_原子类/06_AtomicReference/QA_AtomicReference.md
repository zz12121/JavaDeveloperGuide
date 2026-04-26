---
title: AtomicReference
tags:
  - Java/并发
  - 问答
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicReference（对象引用的原子更新，CAS + volatile）

## Q1：AtomicReference 和 AtomicInteger 有什么区别？

**A**：

| 维度 | AtomicInteger | AtomicReference |
|------|--------------|-----------------|
| 操作类型 | int 基本类型 | 对象引用 |
| CAS 比较 | 值相等（==） | 引用相等（==） |
| 内部存储 | volatile int | volatile Object |
| 适用场景 | 计数器、状态码 | 无锁数据结构、不可变对象 |

`AtomicReference` 的 `compareAndSet` 比较的是引用地址（`==`），不是 `equals()`。

---

## Q2：如何用 AtomicReference 实现无锁栈？

**A**：

```java
class Node<E> { E item; Node<E> next; }
AtomicReference<Node<E>> top = new AtomicReference<>();

// push：头插法
void push(E item) {
    Node<E> newNode = new Node<>(item);
    do {
        newNode.next = top.get();  // 新节点指向当前栈顶
    } while (!top.compareAndSet(newNode.next, newNode)); // CAS 栈顶
}

// pop：移除栈顶
E pop() {
    Node<E> oldTop;
    do {
        oldTop = top.get();
        if (oldTop == null) return null;
    } while (!top.compareAndSet(oldTop, oldTop.next)); // CAS 栈顶
    return oldTop.item;
}
```

核心思想：每次修改栈顶都用 CAS，失败则重试（自旋）。

---

## Q3：AtomicReference 如何解决 CAS 只能操作一个变量的问题？

**A**：将多个变量封装为一个**不可变对象**，然后整体 CAS 替换：

```java
// 多个变量需要一起原子更新
record Pair(int x, int y) {}  // JDK16+ 不可变

AtomicReference<Pair> ref = new AtomicReference<>(new Pair(0, 0));

// 原子更新 x 和 y
Pair old, next;
do {
    old = ref.get();
    next = new Pair(old.x + 1, old.y + 2);
} while (!ref.compareAndSet(old, next));
```

