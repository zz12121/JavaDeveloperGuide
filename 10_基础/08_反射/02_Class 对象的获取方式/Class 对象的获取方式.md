---
title: Class 对象的获取方式
tags:
  - Java/反射
  - 原理型
module: 08_反射
created: 2026-04-18
---

# Class 对象的获取方式

## 核心结论

获取 `Class` 对象有三种方式：`类名.class`、`Class.forName("全限定名")`、`对象.getClass()`。其中 `类名.class` 在编译期确定，`forName()` 在运行时动态加载，`getClass()` 在运行时获取。

## 深度解析

### 三种获取方式

```java
// 1. 类名.class — 编译时确定，最安全高效
Class<String> clazz1 = String.class;

// 2. Class.forName("全限定类名") — 运行时动态加载，最灵活
Class<?> clazz2 = Class.forName("java.lang.String");

// 3. 对象.getClass() — 运行时获取实际类型
String str = "hello";
Class<?> clazz3 = str.getClass();
```

### 三种方式的区别

| 方式 | 加载时机 | 适用场景 | 特点 |
|------|---------|---------|------|
| `类名.class` | 编译时确定 | 已知具体类 | 不会触发类初始化 |
| `Class.forName()` | 运行时动态加载 | 配置文件/动态加载 | **会触发类初始化**（执行 static 块） |
| `对象.getClass()` | 运行时获取 | 已有实例对象 | 返回对象的**实际运行时类型** |

### getClass() 返回的是运行时类型

```java
// 多态场景
Animal animal = new Dog();
Class<?> clazz = animal.getClass();
System.out.println(clazz); // class Dog（不是 Animal）
```

### Class.forName() 的类初始化行为

```java
// Class.forName() 会触发类的初始化（static 块执行）
Class.forName("com.example.MyClass"); // 触发 static 代码块

// 类名.class 不会触发初始化
MyClass.class; // 不触发 static 代码块
```

> JDBC 加载驱动就是利用了 `Class.forName("com.mysql.cj.jdbc.Driver")` 触发 Driver 的 static 块完成注册。

### 基本类型的 Class 对象

```java
// 基本类型也有 Class 对象
Class<int> intClass = int.class;
Class<double> doubleClass = double.class;

// 包装类
Class<Integer> integerClass = Integer.class;
System.out.println(int.class == Integer.TYPE); // true
```

## 关联知识点
