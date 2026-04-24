---
title: Arrays 工具类常用方法
tags:
  - 原理型
  - Java/其他核心概念
  - 问答
module: 12_其他核心概念
created: 2026-04-18
---

# Arrays 工具类常用方法

## Q1：Arrays.asList() 返回的 List 有什么特点？

**A**：`Arrays.asList()` 返回的是 `java.util.Arrays.ArrayList`（内部类），**不是 `java.util.ArrayList`**。关键特点：
1. **固定大小**：不支持 `add()`/`remove()`，否则抛 `UnsupportedOperationException`
2. **与原数组共享数据**：修改原数组会影响 List，反之亦然
3. **基本类型陷阱**：`Arrays.asList(new int[]{1,2,3})` 返回的是 `List<int[]>`（一个元素），不是 `List<Integer>`

```java
// 正确方式
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
List<Integer> list2 = Arrays.stream(new int[]{1,2,3}).boxed().collect(Collectors.toList());
```

---

## Q2：Arrays.sort() 用的是什么排序算法？

**A**：
- 基本类型（`int[]`/`double[]` 等）：**双轴快速排序（Dual-Pivot Quicksort）**，平均 O(n log n)
- 对象类型（`String[]` 等）：**TimSort**（归并排序 + 插入排序的混合），稳定排序，O(n log n)
- `parallelSort()`：使用 ForkJoinPool 并行排序，适合大数据量

---

## Q3：Arrays.copyOf() 和 System.arraycopy() 有什么区别？

**A**：`Arrays.copyOf()` 底层就是调用 `System.arraycopy()`，但提供了更便捷的 API：

| 对比项 | `System.arraycopy()` | `Arrays.copyOf()` |
|--------|---------------------|-------------------|
| 参数 | 需要指定源和目标数组 | 只需源数组和新长度 |
| 返回值 | void（拷贝到已存在的目标数组） | 返回新数组 |
| 灵活性 | 可指定起始位置 | 可自动扩容或截断 |
```java
// System.arraycopy
int[] dest = new int[3];
System.arraycopy(src, 0, dest, 0, 3);

// Arrays.copyOf（更简洁）
int[] dest = Arrays.copyOf(src, 3);
```

---

## Q4：为什么 int[] 直接打印输出的是地址？

**A**：数组没有重写 `toString()` 方法，默认继承 `Object.toString()`，输出的是 `类名@哈希码`。要查看内容需要用：
```java
Arrays.toString(arr);       // 一维数组：[1, 2, 3]
Arrays.deepToString(matrix); // 多维数组：[[1, 2], [3, 4]]
```

---

## Q5：Arrays.binarySearch() 使用时有什么前提条件？

**A**：数组**必须已排序**，否则结果无意义。找不到时返回 `-(insertion point) - 1`（负数表示未找到，绝对值减一为应该插入的位置）。
```java
int[] arr = {1, 3, 5, 7, 9};
Arrays.binarySearch(arr, 5);   // 2（找到，返回索引）
Arrays.binarySearch(arr, 4);   // -3（应插入位置为 2，返回 -(2) - 1 = -3）
```
