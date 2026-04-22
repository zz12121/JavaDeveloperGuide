---
title: 泛型方法面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型方法

## Q1：如何定义泛型方法？和泛型类有什么区别？

**A：**
泛型方法在**返回值类型前**声明类型参数：
```java
public <T> T identity(T t) { return t; }
```

与泛型类的区别：
- 泛型类的类型参数属于整个实例；泛型方法的类型参数**仅属于该方法**
- 泛型方法可以是 `static`；泛型类的类型参数不能用于静态方法
- 泛型方法调用时**自动推断**类型，无需显式指定

---

## Q2：泛型方法可以是静态方法吗？

**A：**
**可以**。这正是泛型方法相对于类型参数的重要优势：

```java
public class Util {
    // ✅ 静态泛型方法自己声明 <T>
    public static <T> List<T> repeat(T item, int times) {
        List<T> list = new ArrayList<>();
        for (int i = 0; i < times; i++) list.add(item);
        return list;
    }
}
```

而泛型类的类型参数（如 `class Box<T>`）不能在静态方法中使用。

---

## Q3：泛型方法如何实现类型安全的集合转换？

**A：**
```java
// 类型安全的 map 操作
public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
    return list.stream().map(mapper).collect(Collectors.toList());
}

// 使用
List<String> names = Arrays.asList("Alice", "Bob");
List<Integer> lengths = map(names, String::length);  // [5, 3]
```

---

## Q4：以下代码有什么问题？如何修复？

```java
public class Util {
    private T value;  // 错误？
    
    public static <T> T create() { return null; }
}
```

**A：**
`private T value` 有问题，因为 `Util` 类本身没有声明类型参数，`T` 未定义。应该：
- 方案1：把 `Util` 改为泛型类 `class Util<T>`
- 方案2：去掉 `private T value`，因为这只是静态工具类

`static <T> T create()` 是合法的，这是**静态泛型方法**，`<T>` 是在方法上独立声明的。
