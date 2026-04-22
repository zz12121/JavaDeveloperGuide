---
title: 泛型约束面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型约束

## Q1：泛型有哪些使用限制？原因是什么？

**A：**
Java 泛型受类型擦除影响，主要限制：

| 限制 | 原因 |
|------|------|
| 不能 `new T()` | 运行时不知道 T 是什么 |
| 不能创建泛型数组 `T[]` | 数组是协变的，泛型是不变的，运行时类型不确定 |
| 不能用 `instanceof T` | 运行时没有 T 的类型信息 |
| 静态成员不能用类型参数 | 类型参数属于实例，静态成员属于类 |
| 不能用基本类型做类型参数 | 泛型擦除后变 Object，基本类型不是 Object |

---

## Q2：如何在泛型方法中创建类型 T 的实例？

**A：**
两种常用方式：

```java
// 方式1：传入 Class 对象（反射）
public <T> T create(Class<T> clazz) throws Exception {
    return clazz.getDeclaredConstructor().newInstance();
}

// 方式2：传入 Supplier 工厂（更优雅，避免反射）
public <T> T create(Supplier<T> factory) {
    return factory.get();
}
// 调用：create(User::new)
```

---

## Q3：为什么不能创建泛型数组？有什么影响？

**A：**
原因：数组是**协变的**（`String[]` 是 `Object[]` 的子类），而泛型是**不变的**（`List<String>` 不是 `List<Object>` 子类）。如果允许创建泛型数组，会绕过类型检查：

```java
// 假设允许（实际不允许）
List<String>[] arr = new ArrayList<String>[10];
Object[] objs = arr;                  // 数组协变，合法
objs[0] = new ArrayList<Integer>();   // 运行时不报错（类型擦除）
String s = arr[0].get(0);            // 运行时 ClassCastException！
```

所以 Java 直接禁止创建泛型数组，防止堆污染。

---

## Q4：`@SuppressWarnings("unchecked")` 什么时候使用？

**A：**
当必须使用未经检查的类型转换（绕过泛型系统时），且程序员能确保类型安全时使用：

```java
@SuppressWarnings("unchecked")
T[] arr = (T[]) new Object[10];  // 程序员保证这里是安全的
```

注意：这只是抑制编译器警告，并不改变运行时行为，滥用可能掩盖真实的类型安全问题。
