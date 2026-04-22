---
title: List 接口面试题
tags:
  - Java/集合框架
  - 原理型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# List 接口

## Q1：List 接口有哪些特点？

**A：**
1. **有序**：元素按插入顺序存储，每个元素有索引
2. **可重复**：允许存储相同的元素
3. **支持索引操作**：`get(index)`、`set(index, element)`、`add(index, element)`

---

## Q2：ArrayList、LinkedList、Vector 的区别？

**A：**

| 维度 | ArrayList | LinkedList | Vector |
|------|-----------|------------|--------|
| 底层 | 动态数组 | 双向链表 | 动态数组 |
| 随机访问 | O(1) | O(n) | O(1) |
| 插入删除 | O(n) | O(1)（头尾） | O(n) |
| 线程安全 | 不安全 | 不安全 | 安全 |
| 扩容 | 1.5 倍 | 无 | 2 倍 |
| 是否推荐 | ✅ 推荐 | 特定场景推荐 | ❌ 已过时 |

---

## Q3：为什么 Vector 被标记为过时？

**A：**
Vector 的所有方法都用 `synchronized` 修饰，锁粒度太大，性能差。现在推荐使用：
- `Collections.synchronizedList()` 包装
- 并发集合如 `CopyOnWriteArrayList`
