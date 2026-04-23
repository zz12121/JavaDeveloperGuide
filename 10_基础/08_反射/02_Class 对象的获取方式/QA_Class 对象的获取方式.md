---
title: Class 对象的获取方式
tags:
  - Java/反射
  - 原理型
  - 问答
module: 08_反射
created: 2026-04-18
---

# Class 对象的获取方式

## Q：获取 Class 对象有哪几种方式？

**A：** 三种方式：
1. **`类名.class`**：编译时确定，不会触发类初始化，最安全高效
   ```java
   Class<String> clazz = String.class;
   ```
2. **`Class.forName("全限定类名")`**：运行时动态加载，**会触发类初始化**（执行 static 块），最灵活
   ```java
   Class<?> clazz = Class.forName("java.lang.String");
   ```
3. **`对象.getClass()`**：运行时获取，返回对象的**实际运行时类型**（不是声明类型）
   ```java
   Animal a = new Dog();
   a.getClass(); // class Dog
   ```

## Q：Class.forName() 和 类名.class 有什么区别？
**A：** 核心区别在于**是否触发类初始化**：
- `Class.forName()` 会触发类的初始化（执行 static 代码块），JDBC 加载驱动就利用了这一点
- `类名.class` 不会触发类初始化，只是加载类而不执行 static 块
- `forName()` 可以动态加载配置文件中指定的类名，`类名.class` 必须在编译时确定

## Q：基本类型有 Class 对象吗？
**A：** 有。每种基本类型都有对应的 Class 对象：`int.class`、`double.class` 等。包装类通过 `Integer.TYPE` 获取对应基本类型的 Class 对象，`int.class == Integer.TYPE` 为 true。
