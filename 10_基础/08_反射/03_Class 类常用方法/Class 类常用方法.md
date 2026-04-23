---
title: Class 类常用方法
tags:
  - Java/反射
  - 原理型
module: 08_反射
created: 2026-04-18
---

# Class 类常用方法

## 核心结论

`Class` 对象是反射的入口，提供了获取类结构信息的全套 API：获取字段、方法、构造器、注解、父类、接口等。方法分为 `getDeclaredXxx()`（获取本类声明的，包括 private）和 `getXxx()`（获取本类及父类的 public 成员）两组。

## 深度解析

### 获取构造器

```java
// 获取所有 public 构造器（含父类）
Constructor<?>[] constructors = clazz.getConstructors();

// 获取所有声明的构造器（包括 private）
Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();

// 获取指定参数类型的 public 构造器
Constructor<User> constructor = clazz.getConstructor(String.class, int.class);

// 获取指定参数类型的构造器（包括 private）
Constructor<User> declared = clazz.getDeclaredConstructor(String.class);
```

### 获取方法

```java
// 获取所有 public 方法（含继承自父类的）
Method[] methods = clazz.getMethods();

// 获取所有声明的方法（本类声明的，包括 private，不含继承的）
Method[] declaredMethods = clazz.getDeclaredMethods();

// 获取指定名称和参数类型的 public 方法
Method method = clazz.getMethod("setName", String.class);

// 获取指定名称和参数类型的方法（包括 private）
Method declared = clazz.getDeclaredMethod("privateMethod");
```

### 获取字段

```java
// 获取所有 public 字段（含继承的）
Field[] fields = clazz.getFields();

// 获取所有声明的字段（包括 private，不含继承的）
Field[] declaredFields = clazz.getDeclaredFields();

// 获取指定名称的 public 字段
Field field = clazz.getField("name");

// 获取指定名称的字段（包括 private）
Field declaredField = clazz.getDeclaredField("privateField");
```

### 其他常用方法

```java
// 类信息
clazz.getName()                    // 全限定类名
clazz.getSimpleName()              // 简单类名
clazz.getPackage()                 // 包信息
clazz.getSuperclass()              // 父类 Class 对象
clazz.getInterfaces()              // 实现的接口数组
clazz.isInterface()                // 是否为接口
clazz.isArray()                    // 是否为数组
clazz.isPrimitive()                // 是否为基本类型
clazz.isEnum()                     // 是否为枚举

// 注解
clazz.getAnnotation(MyAnno.class)  // 获取指定注解
clazz.getAnnotations()             // 获取所有注解

// 实例化
clazz.newInstance()                // 弃用，推荐 getDeclaredConstructor().newInstance()

// 资源
clazz.getResourceAsStream("config.properties")
```

### getDeclaredXxx() vs getXxx()

| 方法 | 范围 | 权限 |
|------|------|------|
| `getMethods()` | 本类 + 继承的 | 仅 public |
| `getDeclaredMethods()` | 仅本类 | 所有权限（含 private） |
| `getFields()` | 本类 + 继承的 | 仅 public |
| `getDeclaredFields()` | 仅本类 | 所有权限（含 private） |

## 关联知识点

