---
title: 注解的提取
tags:
  - Java/注解
  - 场景型
  - 问答
module: 07_注解
created: 2026-04-18
---

# 注解的提取

## Q：如何通过反射获取注解信息？

**A：** 通过 Class、Method、Field 等反射对象的方法获取：
```java
// 判断是否存在某注解
boolean present = clazz.isAnnotationPresent(Log.class);

// 获取指定类型的注解
Log log = clazz.getAnnotation(Log.class);

// 获取所有注解（含继承的）
Annotation[] all = clazz.getAnnotations();

// 获取直接声明的注解（不含继承的）
Annotation[] declared = clazz.getDeclaredAnnotations();
```
前提是注解的 `@Retention(RetentionPolicy.RUNTIME)`，否则运行时读取返回 null。

## Q：getAnnotations() 和 getDeclaredAnnotations() 的区别？
**A：**
- `getAnnotations()`：获取所有注解，包括通过 `@Inherited` 从父类继承的注解
- `getDeclaredAnnotations()`：只获取当前元素直接声明的注解，不包括继承的

## Q：如何获取方法参数上的注解？
**A：** JDK 8 引入了 `java.lang.reflect.Parameter`，可以通过 `method.getParameters()` 获取参数列表，再调用 `param.getAnnotation()` 获取参数上的注解。注意编译时需要加上 `-parameters` 参数才能保留参数名，否则只能拿到 arg0、arg1 这样的占位名。
