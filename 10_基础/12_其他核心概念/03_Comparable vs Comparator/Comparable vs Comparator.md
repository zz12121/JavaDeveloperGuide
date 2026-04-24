---
title: Comparable与Comparator
tags:
  - 对比型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

- **Comparable**（内部比较器）：在类内部实现，定义**自然排序**规则
- **Comparator**（外部比较器）：在类外部定义，支持**多种排序**策略

---

## 深度解析

### 1. Comparable

```java
// 实现接口，定义自然排序
public class User implements Comparable<User> {
    private String name;
    private int age;

    @Override
    public int compareTo(User other) {
        return this.age - other.age; // 按年龄升序
        // 返回负数：this < other
        // 返回 0：相等
        // 返回正数：this > other
    }
}

// 使用
Collections.sort(list);          // 自然排序
list.stream().sorted();          // Stream 自然排序
new TreeSet<>(list);             // TreeSet 自然排序
```

### 2. Comparator

```java
// 方式一：匿名内部类
Collections.sort(list, new Comparator<User>() {
    @Override
    public int compare(User a, User b) {
        return a.getAge() - b.getAge();
    }
});

// 方式二：Lambda
Collections.sort(list, (a, b) -> a.getAge() - b.getAge());

// 方式三：Comparator 静态方法
Collections.sort(list, Comparator.comparingInt(User::getAge));

// 方式四：链式组合
Comparator<User> cmp = Comparator
    .comparing(User::getAge)           // 先按年龄
    .thenComparing(User::getName);     // 年龄相同按姓名

// 常用方法
Comparator.comparingInt(User::getAge)  // 基本类型避免装箱
Comparator.reverseOrder()              // 逆序
Comparator.naturalOrder()              // 自然顺序
comparator.reversed()                  // 某个比较器取反
```

### 3. 对比

| 维度 | Comparable | Comparator |
|------|------------|------------|
| 包 | java.lang | java.util |
| 方法 | `compareTo(T)` | `compare(T1, T2)` |
| 位置 | 类内部实现 | 类外部定义 |
| 排序规则数 | 只有一种（自然排序） | 可定义多种 |
| 修改类 | 需要修改源码 | 不需要修改源码 |
| 使用场景 | 类有明确的自然排序 | 需要多种排序策略 |

### 4. compareTo 的返回值约定

```java
// 标准写法（避免整数溢出）
@Override
public int compareTo(User other) {
    return Integer.compare(this.age, other.age); // 而非 this.age - other.age
}

// Integer.compare 内部实现安全：
// return (x < y) ? -1 : ((x == y) ? 0 : 1);
```
> **陷阱**：`a - b` 在极端值下会溢出。如 `Integer.MIN_VALUE - 1` 结果为正数，排序错误。

---

## 关联知识点
