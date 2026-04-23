---
title: 85 元注解
tags:
  - Java/注解
  - 原理型
  - 问答
module: 07_注解
created: 2026-04-18
---

# 元注解

## Q：什么是元注解？Java 有哪些元注解？

**A：** 元注解是用于修饰注解的注解。Java 标准库提供了 5 个元注解：

1. `@Target` — 指定注解可以用在哪里（类、方法、字段等）
2. `@Retention` — 指定注解的保留策略（SOURCE/CLASS/RUNTIME）
3. `@Documented` — 指定注解是否出现在 JavaDoc 中
4. `@Inherited` — 指定注解是否可以被子类继承
5. `@Repeatable`（JDK 8+） — 指定注解是否可以在同一位置重复使用

## Q：@Retention 的三种策略有什么区别？

**A：**
- `SOURCE`：注解只存在于源码中，编译后丢弃，仅用于编译器检查（如 `@Override`）
- `CLASS`：注解保留到 class 字节码文件中，但 JVM 运行时不可通过反射读取（注解的默认策略）
- `RUNTIME`：注解保留到运行时，可通过反射读取（Spring 框架的自定义注解都是 RUNTIME）

## Q：@Inherited 的继承机制是怎样的？

**A：** `@Inherited` 使注解可以被子类继承：
- 只对 `@Target(ElementType.TYPE)` 类型的注解生效
- 子类通过 `extends` 继承父类时自动获得注解
- 子类如果自行标注了同名注解，则覆盖父类继承的注解
- 对接口的实现、方法的覆盖不生效

## Q：JDK 8 的 @Repeatable 解决了什么问题？

**A：** JDK 8 之前，同一位置不能重复使用相同注解。`@Repeatable` 允许在同一位置重复标注同一注解。需要同时定义一个**容器注解**，编译器在重复使用时会自动将多个注解放入容器注解中。
