---
title: JDK 内置注解
tags:
  - Java/注解
  - 原理型
  - 问答
module: 07_注解
created: 2026-04-18
---

# JDK 内置注解

## Q：JDK 有哪些内置注解？

**A：** JDK 内置了 7 个注解：
- `java.lang` 包：`@Override`、`@Deprecated`、`@SuppressWarnings`
- `java.lang.annotation` 包：`@Target`、`@Retention`、`@Documented`、`@Inherited`（这 4 个是元注解）

## Q：@Override 和 @Deprecated 的区别？

**A：**
- `@Override`：标记方法重写了父类方法，编译期检查，保留策略为 SOURCE
- `@Deprecated`：标记元素已过时，编译时产生警告，保留策略为 RUNTIME，IDE 会显示删除线

## Q：@SuppressWarnings 常用的参数有哪些？

**A：**
- `unchecked`：抑制未检查的类型转换警告（泛型擦除场景最常用）
- `deprecation`：抑制使用过时 API 的警告
- `rawtypes`：抑制原始类型警告
- `unused`：抑制未使用变量/方法的警告

## Q：@Override 注解能不能用在类上？

**A：** 不能。`@Override` 的 `@Target` 只包含 `ElementType.METHOD`。
