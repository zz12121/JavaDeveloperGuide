---
title: ArrayList vs LinkedList
tags:
  - Java/集合框架
  - 对比型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# ArrayList vs LinkedList

## Q1：ArrayList 和 LinkedList 的区别？

**A：**

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 插入删除 | O(n)（中间） | O(n) 遍历 + O(1) 操作 |
| 内存 | 连续内存，缓存友好 | 节点分散，额外指针开销 |
| 接口 | List | List + Deque |

---

## Q2：实际开发中应该选哪个？

**A：**
绝大多数场景选 **ArrayList**。原因：
1. CPU 缓存局部性使得数组连续访问更快
2. 尾部追加都是均摊 O(1)，ArrayList 无对象创建开销
3. 中间操作两者都是 O(n)，ArrayList 数组拷贝反而更快

只有在频繁头部操作时才考虑 `ArrayDeque`（而非 LinkedList）。

---

## Q3：为什么 ArrayList 在实际测试中往往比 LinkedList 更快？

**A：**
1. **CPU 缓存**：数组在内存中连续存储，CPU 缓存命中率高
2. **对象开销**：LinkedList 每插入一个元素需要 `new Node()`，有对象分配开销
3. **GC 压力**：LinkedList 产生更多短生命周期对象
