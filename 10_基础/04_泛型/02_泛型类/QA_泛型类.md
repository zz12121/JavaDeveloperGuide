---
title: 泛型类面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型类

## Q1：如何定义一个泛型类？

**A：**
在类名后用尖括号声明类型参数：
```java
public class Box<T> {
    private T value;
    public T getValue() { return value; }
}
// 使用
Box<String> box = new Box<>();
```

---

## Q2：泛型类的类型参数可以有约束吗？

**A：**
可以，使用 `extends` 关键字设置上界（Java 泛型只有上界，没有下界约束）：

```java
// 单个上界
public class NumBox<T extends Number> { ... }

// 多个上界（类必须放第一个，接口在后）
public class MultiBox<T extends Comparable<T> & Cloneable> { ... }
```

---

## Q3：为什么泛型类的静态方法不能使用类的类型参数？

**A：**
因为静态成员属于**类级别**，在 JVM 中只有一份，而泛型类型参数属于**实例级别**（每个实例的 T 可以不同）。静态方法无法确定 T 的具体类型，因此编译器禁止：

```java
public class Box<T> {
    private T value;
    
    // ❌ 编译错误：静态方法不能引用类的类型参数 T
    public static void print(T t) { ... }
    
    // ✅ 正确：静态泛型方法自己声明类型参数
    public static <U> void print(U u) { ... }
}
```

---

## Q4：泛型类可以继承普通类或其他泛型类吗？

**A：**
可以，有以下几种形式：
```java
// 继承普通类
class MyList<T> extends AbstractList<T> { ... }

// 固定父类类型参数
class StringBox extends Box<String> { ... }

// 传递自己的类型参数给父类
class WrapBox<T> extends Box<T> { ... }

// 扩展额外类型参数
class DetailBox<T, U> extends Box<T> { ... }
```
