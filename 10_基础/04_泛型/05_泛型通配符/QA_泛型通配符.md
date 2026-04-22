---
title: 泛型通配符
tags:
  - Java/泛型
  - 对比型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型通配符

## Q1：`? extends T` 和 `? super T` 的区别是什么？

**A：**

| | `? extends T` | `? super T` |
|--|---------------|-------------|
| 含义 | T 或 T 的子类 | T 或 T 的父类 |
| 读取 | 可以，返回 T | 只能返回 Object |
| 写入 | 不能（除 null） | 可以，写入 T 及子类 |
| 语义 | 生产者（Producer） | 消费者（Consumer） |
| 记忆口诀 | PECS：Producer Extends | PECS：Consumer Super |

---

## Q2：为什么 `List<? extends Number>` 不能 add？

**A：**
因为编译器**不知道实际是哪种子类**：

```java
List<? extends Number> list = new ArrayList<Integer>();
// list.add(1.0);  // 不允许——list 实际上是 List<Integer>，不能放 Double
// list.add(1);    // 也不允许——编译器无法确认，统一禁止（除 null）
```

只要允许写入，就可能破坏类型安全。编译器宁可全部禁止写入，保证安全。

---

## Q3：`List<Object>` 和 `List<?>` 有什么区别？

**A：**
```java
List<String> strings = new ArrayList<>();

// List<Object> 不兼容 List<String>
List<Object> objects = strings;  // ❌ 编译错误

// List<?> 可以接收任意类型的 List
List<?> wildcards = strings;     // ✅
```

区别：
- `List<Object>`：元素类型必须是 `Object`（精确），可以 `add(Object)`
- `List<?>`：元素类型未知，**不能 add**（除 null），只能读取为 Object

---

## Q4：以下哪个方法签名更通用？为什么？

```java
// 方法 A
void copy(List<Object> dest, List<Object> src) { ... }

// 方法 B
void copy(List<? super T> dest, List<? extends T> src) { ... }
```

**A：**
**方法 B 更通用**。

方法 A 只能接收 `List<Object>`，`List<String>` 等都不能传入。

方法 B（PECS 原则）：
- `src` 是 `? extends T`（生产者，只读），可接收 `List<Integer>`、`List<Double>` 等
- `dest` 是 `? super T`（消费者，可写），可接收 `List<Number>`、`List<Object>` 等

这正是 `Collections.copy` 方法的签名。
