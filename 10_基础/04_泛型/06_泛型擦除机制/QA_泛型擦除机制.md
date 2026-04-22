---
title: 泛型擦除机制面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型擦除机制

## Q1：什么是泛型擦除？擦除后类型参数变成什么？

**A：**
泛型擦除是指 Java 编译器在生成字节码时，将所有泛型类型参数信息移除：
- 无界 `T` → `Object`
- 有上界 `T extends Number` → `Number`
- 多界 `T extends A & B` → `A`（第一个上界）

```java
// 源码 Box<T>  擦除后 → Box（T 变 Object）
// 源码 Box<T extends Number>  擦除后 → Box（T 变 Number）
```

---

## Q2：为什么 Java 要设计类型擦除？

**A：**
历史原因：JDK 5 引入泛型时，为了**向后兼容**——让泛型代码可以在旧 JVM 上运行，同时旧代码也能与新的泛型类交互，设计了类型擦除方案。

代价：失去了运行时类型信息，带来了一系列约束（不能 `new T()`、`T[]`、`instanceof T` 等）。

---

## Q3：`List<String>` 和 `List<Integer>` 可以方法重载吗？

**A：**
**不能**。由于类型擦除，两者在运行时都是 `List`，签名相同，构成重复定义而不是重载：

```java
// ❌ 编译错误：方法签名冲突
public void process(List<String> list) { ... }
public void process(List<Integer> list) { ... }
```

---

## Q4：如何在运行时获取泛型类型参数？

**A：**
利用**匿名子类 + `getGenericSuperclass()`**（超类类型令牌）：

```java
// 创建匿名子类，泛型信息会被写入字节码
ParameterizedType type = (ParameterizedType) 
    new ArrayList<String>(){}.getClass().getGenericSuperclass();
Type typeArg = type.getActualTypeArguments()[0];
System.out.println(typeArg);  // class java.lang.String
```

这就是 Jackson 的 `TypeReference`、Gson 的 `TypeToken` 的实现原理，用来反序列化泛型类型。

---

## Q5：以下代码为什么可以绕过泛型检查？

```java
List<Integer> intList = new ArrayList<>();
List rawList = intList;
rawList.add("string");  // 运行时不报错
System.out.println(intList.get(0));  // ClassCastException
```

**A：**
由于类型擦除，`rawList.add("string")` 在运行时操作的是 `ArrayList`（没有类型约束），所以不报错。但 `intList.get(0)` 时，编译器插入了 `(Integer)` 强转，实际是 String，所以抛 `ClassCastException`。

这说明：**绕过泛型的编译期检查，可能导致运行时 ClassCastException。**
