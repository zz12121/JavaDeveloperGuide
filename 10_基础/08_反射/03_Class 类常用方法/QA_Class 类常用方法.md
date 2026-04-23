---
title: Class 类常用方法
tags:
  - Java/反射
  - 原理型
  - 问答
module: 08_反射
created: 2026-04-18
---

# Class 类常用方法

## Q：Class 类常用的获取字段、方法、构造器的方法有哪些？

**A：** 分为两组：

**getXxx() 系列**（获取 public 成员，含继承的）：
- `getFields()` / `getField(name)` — public 字段
- `getMethods()` / `getMethod(name, paramTypes)` — public 方法
- `getConstructors()` / `getConstructor(paramTypes)` — public 构造器

**getDeclaredXxx() 系列**（获取本类所有成员，含 private，不含继承）：
- `getDeclaredFields()` / `getDeclaredField(name)` — 所有字段
- `getDeclaredMethods()` / `getDeclaredMethod(name, paramTypes)` — 所有方法
- `getDeclaredConstructors()` / `getDeclaredConstructor(paramTypes)` — 所有构造器

## Q：getMethods() 和 getDeclaredMethods() 有什么区别？

**A：**
- `getMethods()`：返回本类及**继承的**所有 public 方法（包括 Object 类的 toString、equals 等）
- `getDeclaredMethods()`：只返回**本类声明**的所有方法（包括 private），不含继承的方法

同理适用于 getFields/getDeclaredFields。

## Q：Class.forName() 和 Class.newInstance() 有什么区别？

**A：** 这是两个完全不同的操作：
- `Class.forName("类名")`：获取 Class 对象（加载+初始化类）
- `Class.newInstance()`：调用无参构造器创建对象（JDK 9 标记为弃用）

创建对象推荐使用 `clazz.getDeclaredConstructor().newInstance()`，能正确处理构造器异常。
