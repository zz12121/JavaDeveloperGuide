---
title: "@Repeatable注解"
tags:
  - Java/注解
  - 问答
  - 原理型
module: 07_注解
created: 2026-04-25
---

# @Repeatable注解

## Q1：@Repeatable 的实现原理是什么？

**A：**
@Repeatable 是**编译期处理**，不是运行时特性。编译器在遇到重复注解时，会自动将其转换为容器注解形式：

```java
// 源代码
@Schedule(fixedDelay = 1000)
@Schedule(fixedDelay = 2000)
public void doSomething() { }

// 编译器自动生成等效于：
@Schedules({
    @Schedule(fixedDelay = 1000),
    @Schedule(fixedDelay = 2000)
})
public void doSomething() { }
```

运行时反射时，`getAnnotationsByType()` 会展开容器中的注解返回数组。

---

## Q2：@Repeatable 和 Spring 的别名模式有什么区别？

**A：**
- `@Repeatable`：JDK 8+ 标准，语法是多个独立注解，可读性更高
- 别名模式（Spring 使用）：使用数组形式，兼容任意 JDK 版本

```java
// @Repeatable 模式（更自然）
@Validation(msg = "错误1")
@Validation(msg = "错误2")
public void process() { }

// 别名模式（Spring 常用）
@Validation({"错误1", "错误2"})
public void process() { }
```

两者效果相似，但 @Repeatable 的语法更接近自然语言。

---

## Q3：@Repeatable 的容器注解必须叫什么名字？

**A：**
没有固定命名约定，但按照惯例使用 `XXXs` 后缀：
- `@Schedule` → `@Schedules`
- `@Authority` → `@Authorities`
- `@Validation` → `@Validations`

容器注解必须有一个名为 `value()` 的方法，返回被包装注解类型的数组：
```java
@interface Schedules {
    Schedule[] value();  // 必须名为 value，返回被包装类型数组
}
```

---

## Q4：如何读取重复注解？

**A：**
```java
Method method = MyTask.class.getMethod("doSomething");

// JDK 8+ 推荐方式
Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
for (Schedule s : schedules) {
    System.out.println(s.fixedDelay());
}

// JDK 8 之前的方式（仍然兼容）
Schedules schedules = method.getAnnotation(Schedules.class);
if (schedules != null) {
    for (Schedule s : schedules.value()) {
        System.out.println(s.fixedDelay());
    }
}
```
