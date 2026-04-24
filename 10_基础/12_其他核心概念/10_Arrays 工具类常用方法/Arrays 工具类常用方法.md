---
title: Arrays 工具类常用方法
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Arrays 工具类常用方法

## 核心结论

`java.util.Arrays` 是 JDK 提供的数组操作工具类，包含排序、查找、填充、拷贝、比较、转换为 List 等常用静态方法。其中 `Arrays.asList()` 返回的是**固定大小的 List**，不能增删，面试高频考点。

---

## 深度解析

### 1. 排序与查找

```java
int[] arr = {5, 3, 1, 4, 2};

// 排序（双轴快速排序，O(n log n)）
Arrays.sort(arr);                    // [1, 2, 3, 4, 5]

// 部分排序（对 [fromIndex, toIndex) 范围排序）
Arrays.sort(arr, 0, 3);

// 并行排序（大数据量时更快，JDK 8）
Arrays.parallelSort(arr);

// 二分查找（数组必须已排序，找不到返回 -(insertion point) - 1）
int idx = Arrays.binarySearch(arr, 3);  // 2

// 对象排序（需实现 Comparable 或传入 Comparator）
String[] strs = {"banana", "apple", "cherry"};
Arrays.sort(strs);                   // [apple, banana, cherry]
Arrays.sort(strs, Comparator.reverseOrder());
```

### 2. 拷贝与填充

```java
int[] src = {1, 2, 3, 4, 5};

// 拷贝指定长度（底层调用 System.arraycopy）
int[] copy1 = Arrays.copyOf(src, 3);       // [1, 2, 3]
int[] copy2 = Arrays.copyOf(src, 7);       // [1, 2, 3, 4, 5, 0, 0]（超长补默认值）

// 指定范围拷贝
int[] copy3 = Arrays.copyOfRange(src, 1, 4); // [2, 3, 4]

// 填充
int[] filled = new int[5];
Arrays.fill(filled, 42);               // [42, 42, 42, 42, 42]
Arrays.fill(filled, 1, 3, 99);        // [42, 99, 99, 42, 42]
```

### 3. 比较与判断

```java
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};

// 比较内容（逐个元素比较）
Arrays.equals(a, b);                   // true

// 深度比较（嵌套数组用这个）
int[][] c = {{1, 2}, {3, 4}};
int[][] d = {{1, 2}, {3, 4}};
Arrays.equals(c, d);                   // false（比较的是引用）
Arrays.deepEquals(c, d);               // true（递归比较内容）
```

### 4. Arrays.asList() — 面试重点 ⚠️

```java
// 基本用法
List<String> list = Arrays.asList("A", "B", "C");

// ⚠️ 陷阱1：返回的是固定大小的 List，不支持增删
list.add("D");       // UnsupportedOperationException
list.remove(0);      // UnsupportedOperationException

// ✅ 正确方式：包装为可变 List
List<String> mutable = new ArrayList<>(Arrays.asList("A", "B", "C"));

// ⚠️ 陷阱2：基本类型数组的问题
int[] nums = {1, 2, 3};
List<int[]> list2 = Arrays.asList(nums);  // List 包含一个 int[] 元素，长度为 1！
// 正确方式
List<Integer> list3 = Arrays.stream(nums).boxed().collect(Collectors.toList());

// ⚠️ 陷阱3：修改数组会影响 List（底层共享同一数组）
String[] arr = {"A", "B", "C"};
List<String> list4 = Arrays.asList(arr);
arr[0] = "X";
System.out.println(list4.get(0));  // "X"（受影响）
```

### 5. toString 与 Stream

```java
int[] arr = {1, 2, 3};

// 数组直接打印是地址，需要用 Arrays.toString
System.out.println(arr);              // [I@xxx
System.out.println(Arrays.toString(arr)); // [1, 2, 3]

// 多维数组用 deepToString
int[][] matrix = {{1, 2}, {3, 4}};
System.out.println(Arrays.deepToString(matrix)); // [[1, 2], [3, 4]]

// 转为 Stream（JDK 8）
IntStream stream = Arrays.stream(arr);
stream.sum();  // 6
```

### 6. 方法分类速查

| 分类 | 方法 | 说明 |
|------|------|------|
| 排序 | `sort()` / `parallelSort()` | 排序 |
| 查找 | `binarySearch()` | 二分查找（需已排序） |
| 拷贝 | `copyOf()` / `copyOfRange()` | 数组拷贝 |
| 填充 | `fill()` | 填充值 |
| 比较 | `equals()` / `deepEquals()` | 内容比较 |
| 转换 | `asList()` / `toString()` / `deepToString()` | 转换 |
| 哈希 | `hashCode()` / `deepHashCode()` | 计算哈希值 |

---

## 关联知识点

