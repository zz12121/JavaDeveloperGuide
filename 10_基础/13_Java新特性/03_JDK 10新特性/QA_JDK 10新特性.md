---
title: JDK 10新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# var 局部变量类型推断

## Q1：var 是什么？Java 是不是变成动态类型语言了？

**A**：不是。`var` 是 JDK 10 引入的**编译期语法糖**，变量类型在编译时就已经确定，运行时与显式声明类型完全一致。它不会让 Java 变成 JavaScript/Python 那样的动态类型语言。`var` 仅用于局部变量，且必须初始化。

---

## Q2：var 可以用在哪些地方？不能用在哪些地方？

**A**：
- ✅ **可用**：局部变量声明（含初始化）、try-with-resources、for 循环增强
- ❌ **不可用**：成员变量、方法参数、方法返回值、Lambda 表达式、未初始化变量、赋值为 null

---

## Q3：var 和 JS/Python 的 var/let 有什么区别？

**A**：本质区别：
- **Java var**：编译期推断，类型固定不变。`var x = "hello"; x = 123;` 会编译错误。
- **JS let**：运行时动态类型，类型可变。`let x = "hello"; x = 123;` 完全合法。

Java var 只是一种省略类型声明的便利写法。

---

## Q4：`var list = new ArrayList<>();` 有什么问题？

**A**：这里 var 会推断为 `ArrayList<Object>`（因为 `<>` 钻石运算符依赖左侧类型信息，但左侧是 var）。如果需要指定泛型，必须显式写出：
```java
var list = new ArrayList<String>();  // 正确，推断为 ArrayList<String>
```

---

## Q5：什么时候应该用 var，什么时候不应该用？

**A**：
- ✅ **推荐用**：类型名很长（如 `Map<String, List<Map.Entry<Integer, String>>>`），用 var 大幅提升可读性
- ✅ **推荐用**：右侧类型明确（如 `var stream = list.stream();`）
- ❌ **不推荐用**：右侧是方法调用且返回类型不直观时，保留显式类型更清晰
