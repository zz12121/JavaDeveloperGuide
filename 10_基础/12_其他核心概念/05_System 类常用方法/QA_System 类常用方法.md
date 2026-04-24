---
title: System类常用方法
tags:
  - Java/其他核心概念
  - 原理型
  - 问答
module: 12_其他核心概念
created: 2026-04-18
---

# System类常用方法
## Q1：System.arraycopy 和 Arrays.copyOf 有什么区别？

**A**：
- **System.arraycopy**：需要目标数组已存在并指定位置，可指定源和目标的起始偏移量
- **Arrays.copyOf**：内部调用 arraycopy，但更简洁——自动创建目标数组，支持截断或扩展
```java
int[] src = {1, 2, 3, 4, 5};

// System.arraycopy：目标数组必须存在
int[] dest = new int[5];
System.arraycopy(src, 0, dest, 0, 3); // dest = [1, 2, 3, 0, 0]

// Arrays.copyOf：自动创建目标数组
int[] copy = Arrays.copyOf(src, 3);  // [1, 2, 3]
int[] ext = Arrays.copyOf(src, 7);   // [1, 2, 3, 4, 5, 0, 0]
```
> 性能上几乎相同，Arrays.copyOf 更方便但灵活性不如 arraycopy。
