---
title: "@FunctionalInterface 注解"
tags:
  - Java/注解
  - 原理型
module: 07_注解
created: 2026-04-18
---

# @FunctionalInterface 注解

## 核心结论

`@FunctionalInterface` 是 JDK 8 引入的标记注解，用于标识一个接口为**函数式接口**（只包含一个抽象方法）。编译器会检查标注了该注解的接口是否符合函数式接口的定义，不符合则编译报错。

## 深度解析

### 什么是函数式接口

函数式接口（Functional Interface）是指**只包含一个抽象方法**的接口。Lambda 表达式的类型就是函数式接口。

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);  // 唯一的抽象方法
}
```

### @FunctionalInterface 的编译期检查

```java
// ✅ 正确：只有一个抽象方法
@FunctionalInterface
public interface MyFunc {
    void run();
}

// ❌ 编译报错：有多个抽象方法，不是函数式接口
@FunctionalInterface
public interface MyFunc2 {
    void run();
    void execute();  // 编译器报错：Multiple non-overriding abstract methods found
}
```

### 允许的额外内容

函数式接口可以包含以下内容（不影响函数式接口的身份）：

```java
@FunctionalInterface
public interface MyFunction {
    // 唯一的抽象方法
    String apply(String input);

    // ✅ Object 类中的 public 方法（不算抽象方法）
    @Override
    String toString();
    @Override
    boolean equals(Object obj);
    @Override
    int hashCode();

    // ✅ default 方法（JDK 8+）
    default void printHello() {
        System.out.println("hello");
    }

    // ✅ static 方法（JDK 8+）
    static MyFunction identity() {
        return s -> s;
    }
}
```

### 为什么需要 @FunctionalInterface

- **不写** `@FunctionalInterface`，只要接口满足条件（只有一个抽象方法），仍然可以作为函数式接口使用
- **写了** `@FunctionalInterface`，编译器会强制检查，防止后续维护时意外增加抽象方法

### JDK 内置的函数式接口（java.util.function）

| 接口 | 抽象方法 | 说明 |
|------|---------|------|
| `Function<T,R>` | `R apply(T t)` | 接收 T，返回 R |
| `Consumer<T>` | `void accept(T t)` | 接收 T，无返回 |
| `Supplier<T>` | `T get()` | 无参数，返回 T |
| `Predicate<T>` | `boolean test(T t)` | 接收 T，返回 boolean |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | 接收两个参数 |

## 代码示例

```java
// 使用 Lambda 表达式
Predicate<String> isNotEmpty = s -> s != null && !s.isEmpty();
Function<Integer, String> intToStr = i -> String.valueOf(i);

// 自定义函数式接口
@FunctionalInterface
public interface Validator<T> {
    boolean validate(T input);

    default Validator<T> and(Validator<T> other) {
        return input -> this.validate(input) && other.validate(input);
    }
}
```

## 关联知识点

