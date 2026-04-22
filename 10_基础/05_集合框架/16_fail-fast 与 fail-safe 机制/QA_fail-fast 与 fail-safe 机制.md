---
title: fail-fast 与 fail-safe 机制
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# fail-fast 与 fail-safe 机制

## Q1：什么是 fail-fast 机制？

**A**：fail-fast（快速失败）是 Java 集合的一种错误检测机制。当遍历集合时，如果其他线程（或本线程通过集合自身方法）对集合进行了**结构性修改**（增删元素），Iterator 会立即抛出 `ConcurrentModificationException`。
原理是集合内部维护 `modCount` 计数器，Iterator 创建时记录 `expectedModCount`，每次 `next()` 前检查两者是否一致。

---

## Q2：什么是 fail-safe 机制？

**A**：fail-safe（安全失败）机制在遍历时不直接操作原集合，而是**遍历副本**。因此对原集合的修改不会影响正在进行的遍历，也不会抛出异常。
典型代表是 `java.util.concurrent` 包下的集合，如 `ConcurrentHashMap`、`CopyOnWriteArrayList`。

---

## Q3：fail-fast 和 fail-safe 有什么区别？

**A**：

| 对比项 | fail-fast | fail-safe |
|--------|-----------|-----------|
| 异常 | 抛出 `ConcurrentModificationException` | 不抛异常 |
| 遍历对象 | 原集合 | 副本 |
| 内存开销 | 低 | 高（需拷贝） |
| 实时性 | 能看到最新修改 | 遍历的是快照 |
| 代表 | `ArrayList`、`HashMap` | `ConcurrentHashMap`、`CopyOnWriteArrayList` |

---

## Q4：遍历 ArrayList 时删除元素，怎样才不会抛 ConcurrentModificationException？

**A**：三种正确方式：
```java
// 1. Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove();
}

// 2. removeIf（JDK 8 推荐）
list.removeIf(s -> s.equals("B"));

// 3. 倒序遍历 + 索引删除（不推荐，代码不够优雅）
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).equals("B")) list.remove(i);
}
```
**绝对不行**的是在 foreach 循环中调用 `list.remove()`。

---

## Q5：fail-fast 是绝对保证的吗？

**A**：**不是**。fail-fast 只是"尽力而为"的检测机制，JDK 文档明确说明它**不能保证**一定抛出异常。在多线程场景下，不同步的情况下仍然可能出现不可预期的行为。
因此，fail-fast 主要用于**检测 bug**（如单线程中错误的遍历修改），而不是作为线程安全的保障。多线程安全需要使用 `ConcurrentHashMap` 等并发集合或显式加锁。
