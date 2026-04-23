---
title: 自定义注解
tags:
  - Java/注解
  - 场景型
  - 问答
module: 07_注解
created: 2026-04-18
---

# 自定义注解

## Q：如何自定义一个注解？

**A：** 使用 `@interface` 关键字定义，并配合元注解指定范围和生命周期：

```java
@Target(ElementType.METHOD)                    // 可用在方法上
@Retention(RetentionPolicy.RUNTIME)            // 运行时可通过反射读取
public @interface Log {
    String value() default "";                  // 属性声明（本质是抽象方法）
}
```

## Q：注解的属性支持哪些类型？

**A：** 注解属性的类型只能是：8 种基本类型、`String`、`Class<?>`、枚举类型、注解类型、以及以上类型的一维数组。不支持 `Object`、集合类型等。

## Q：为什么注解属性名叫 value 时可以省略属性名？

**A：** 这是 Java 的语法约定。当注解只有一个属性，或需要特别指定的属性名为 `value` 时，使用时可以省略 `value =`：

```java
@Log("查询用户")    // 等价于 @Log(value = "查询用户")
@Log(value = "查询用户", enable = true)  // 有其他属性时不能省略
```

## Q：自定义注解不写 @Retention 会怎样？

**A：** 默认是 `RetentionPolicy.CLASS`，注解会保留到 class 文件中但运行时无法通过反射读取。如果需要在运行时通过反射获取注解信息，必须显式设置为 `RUNTIME`。
