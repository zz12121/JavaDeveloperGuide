---
id: qa_37
title: 泛型的本质面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型的本质

## Q1：什么是泛型？为什么需要泛型？

**A：**
泛型是 JDK 5 引入的**类型参数化**机制，允许将类型本身作为参数传递给类、接口或方法。

需要泛型的原因：
1. **类型安全**：没有泛型时，集合存放 `Object`，存取时需要强转，可能引发运行时 `ClassCastException`；泛型将类型检查提前到编译期
2. **消除强转**：泛型让编译器自动插入类型转换，代码更简洁
3. **代码复用**：同一算法/数据结构（如 `List`、`Stack`）可适配任意类型

---

## Q2：Java 泛型是真泛型还是伪泛型？为什么这么说？

**A：**
Java 泛型是**伪泛型**（或称"类型擦除泛型"）。

原因：Java 的泛型信息只存在于编译期，编译器完成类型检查后，会把泛型信息**擦除**，生成的字节码中没有任何泛型类型参数，类型参数被替换为 `Object` 或其上界。

对比 C# 的泛型是真泛型，运行时依然保留类型信息（reified generics）。

---

## Q3：`List<String>` 和 `List<Integer>` 在运行时是同一个类吗？

**A：**
是同一个类。由于类型擦除，运行时两者都是 `ArrayList`，没有区别：

```java
List<String> ls = new ArrayList<>();
List<Integer> li = new ArrayList<>();
System.out.println(ls.getClass() == li.getClass());  // true
```

这也是为什么 Java 中无法进行 `new T()`、`T[]` 等操作——运行时根本不知道 T 是什么。

---

## Q4：泛型有哪几种使用形式？

**A：**
- **泛型类**：`class Pair<A, B> { ... }`
- **泛型接口**：`interface Repository<T, ID> { ... }`
- **泛型方法**：`<T extends Comparable<T>> T max(T a, T b)`
- **通配符**：`List<? extends Number>`、`List<? super Integer>`
