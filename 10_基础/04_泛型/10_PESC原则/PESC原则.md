---
title: PECS 原则
tags:
  - Java/泛型
  - 原理型
module: 04_泛型
created: 2026-04-18
---

# PECS 原则（Producer Extends, Consumer Super）

## 核心口诀

> **PECS：Producer Extends, Consumer Super**
> - **生产者（读数据）→ 用 `? extends T`**
> - **消费者（写数据）→ 用 `? super T`**

## 为什么需要 PECS

```java
// 问题：想写一个通用的复制方法
// 方案1：只接受同类型，太严格
void copy(List<Number> dest, List<Number> src) { ... }
// 无法传入 List<Integer>、List<Double>

// 方案2：PECS 原则，最灵活
<T> void copy(List<? super T> dest, List<? extends T> src) { ... }
// src 读数据（生产者）→ extends；dest 写数据（消费者）→ super
```

## 详细解析

### Producer Extends（生产者用 extends）

```java
// 只需要读取数据的场景 → ? extends T
public double sum(List<? extends Number> list) {
    double result = 0;
    for (Number n : list) {           // 读取，返回 Number 类型
        result += n.doubleValue();
    }
    return result;
    // list.add(1.0);  ❌ 不能写入
}

// 可以传入：
sum(new ArrayList<Integer>());   // ✅
sum(new ArrayList<Double>());    // ✅
sum(new ArrayList<Long>());      // ✅
```

### Consumer Super（消费者用 super）

```java
// 只需要写入数据的场景 → ? super T
public void addNumbers(List<? super Integer> list) {
    list.add(1);                 // 写入 Integer：✅
    list.add(2);
    // Integer n = list.get(0); ❌ 读取只能得 Object
}

// 可以传入：
addNumbers(new ArrayList<Integer>());   // ✅
addNumbers(new ArrayList<Number>());    // ✅
addNumbers(new ArrayList<Object>());    // ✅
```

## JDK 标准库中的 PECS

```java
// Collections.copy 源码（典型 PECS 应用）
public static <T> void copy(
    List<? super T> dest,       // 消费者：写入
    List<? extends T> src       // 生产者：读取
) { ... }

// 使用示例
List<Integer> src = Arrays.asList(1, 2, 3);
List<Number> dest = new ArrayList<>(Arrays.asList(0.0, 0.0, 0.0));
Collections.copy(dest, src);  // ✅ Integer extends Number

// Stream.collect
// Collectors.toList() 内部也用了 ? super T 做消费者
```

## 决策流程

```
问自己：这个集合是用来做什么的？
  ├── 只读（从集合取数据）      → 用 ? extends T
  ├── 只写（往集合放数据）      → 用 ? super T
  └── 既读又写（或不需要灵活性）→ 用具体类型 T
```

## 经典对比

| 场景 | 通配符 | 能读吗 | 能写吗 |
|------|--------|--------|--------|
| 只读（生产者） | `? extends T` | ✅（返回 T） | ❌ |
| 只写（消费者） | `? super T` | ⚠️（只返回 Object） | ✅ |
| 既读又写 | `T`（具体类型） | ✅ | ✅ |
| 完全未知 | `?` | ⚠️（只返回 Object） | ❌ |

## 关联知识点
