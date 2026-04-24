---
title: Comparable与Comparator
tags:
  - 对比型
  - 问答
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Comparable与Comparator
## Q1：Comparable 和 Comparator 的区别？

**A**：
- **Comparable**：类内部实现 `compareTo()` 方法，定义自然排序（一种规则）
- **Comparator**：类外部定义，实现 `compare()` 方法，支持多种排序策略

| 维度 | Comparable | Comparator |
|------|------------|------------|
| 包 | java.lang | java.util |
| 排序规则 | 一种（自然排序） | 多种 |
| 是否需修改源类 | 是 | 否 |
```java
// Comparable：自然排序
list.sort(null); // 或 Collections.sort(list)

// Comparator：自定义排序
list.sort(Comparator.comparing(User::getAge).reversed());
```

---

## Q2：compareTo 为什么要用 Integer.compare 而不是 a - b？

**A**：`a - b` 在极端值下会**整数溢出**，导致排序结果错误。
```java
int a = Integer.MIN_VALUE;
int b = 1;
a - b; // 溢出！结果为 Integer.MAX_VALUE（正数），排序错误

// 安全写法
Integer.compare(a, b); // 正确返回 -1
```
> **规范**：所有 compareTo 实现都应使用包装类的 `compare` 方法，避免减法溢出。
