---
title: ArrayList vs LinkedList
tags:
  - Java/集合框架
  - 对比型
module: 05_集合框架
created: 2026-04-18
---

# ArrayList vs LinkedList 区别与选择

## 全面对比

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 动态数组 `Object[]` | 双向链表 `Node` |
| 随机访问 `get(i)` | **O(1)** 数组下标 | O(n) 遍历链表 |
| 尾部追加 `add(e)` | 均摊 **O(1)**（偶尔扩容） | **O(1)** |
| 头部插入 `add(0, e)` | O(n) 数组元素后移 | **O(1)** 修改指针 |
| 中间插入 `add(i, e)` | O(n) | O(n) 遍历 + O(1) 插入 |
| 删除 `remove(i)` | O(n) 元素前移 | O(n) 遍历 + O(1) 删除 |
| 内存占用 | 紧凑，但有预留空间浪费 | 每个节点额外 2 个指针（8/16 字节） |
| CPU 缓存 | **友好**（连续内存） | 不友好（节点分散） |
| `RandomAccess` | ✅ 实现了 | ❌ 没实现 |
| 额外接口 | 仅 `List` | `List` + `Deque`（可当栈/队列） |

## 性能误区

### 误区 1：LinkedList 插入删除一定比 ArrayList 快

- **头部插入删除**：LinkedList O(1) > ArrayList O(n) ✅ 正确
- **尾部插入删除**：ArrayList 均摊 O(1) ≈ LinkedList O(1)，ArrayList 通常更快（无对象创建开销）
- **中间插入删除**：两者都是 O(n)，ArrayList 的数组拷贝是连续内存操作，CPU 缓存友好，**实际可能更快**

### 误区 2：LinkedList 内存开销小

LinkedList 每个节点需要额外存储 2 个指针（32 位 JVM 每个 8 字节，64 位压缩指针也是 8 字节），加上对象头开销，**内存占用约为 ArrayList 的 1.5~2 倍**。

## 选择指南

```
需要随机访问多？     → ArrayList
需要频繁尾部追加？   → ArrayList（更优）
需要频繁头部操作？   → LinkedList（或 ArrayDeque）
需要当栈/双端队列？  → LinkedList 或 ArrayDeque（推荐 ArrayDeque）
数据量已知？         → ArrayList + 指定初始容量
```

## 实际开发建议

| 场景 | 推荐 |
|------|------|
| 默认选择 | **ArrayList** |
| 需要栈 | `ArrayDeque`（比 LinkedList+Stack 更快） |
| 需要队列 | `ArrayDeque` 或 `LinkedList` |
| 频繁头部插入删除 | `ArrayDeque`（循环数组，性能优于 LinkedList） |

> 实测中，由于 CPU 缓存局部性原理，ArrayList 在大多数场景下性能优于 LinkedList，即使是在"LinkedList 更优"的头部操作场景，`ArrayDeque`（循环数组）往往也比 LinkedList 更快。

## 关联知识点
