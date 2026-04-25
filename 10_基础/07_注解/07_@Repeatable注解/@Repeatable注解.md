---
title: "@Repeatable注解"
tags:
  - Java/注解
  - JDK8
  - 元注解
module: 07_注解
created: 2026-04-25
---

# @Repeatable注解

## 背景

JDK 8 之前，同一个注解在同一个元素上不能重复使用。开发者被迫使用数组形式的容器注解来曲线救国。JDK 8 引入了 `@Repeatable`，允许直接重复使用注解。

## @Repeatable 定义

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Repeatable {
    Class<? extends Annotation> value();
}
```

## 使用步骤

### 第一步：定义容器注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Schedules {
    Schedule[] value();  // 必须是注解类型数组
}
```

### 第二步：定义可重复注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(Schedules.class)  // 指定容器类型
public @interface Schedule {
    String cron() default "";
    long fixedDelay() default 0;
}
```

### 第三步：使用

```java
@Schedule(fixedDelay = 1000)
@Schedule(fixedDelay = 2000)
@Schedule(cron = "0 0 * * * ?")
public void doSomething() { }
```

## 编译器处理过程

编译器将重复注解自动转换为容器注解形式：

```java
// 源代码
@Schedule(fixedDelay = 1000)
@Schedule(fixedDelay = 2000)
public void doSomething() { }

// 编译后等效于：
@Schedules({
    @Schedule(fixedDelay = 1000),
    @Schedule(fixedDelay = 2000)
})
public void doSomething() { }
```

## 读取重复注解

```java
Method method = MyTask.class.getMethod("doSomething");

// JDK 8+ 推荐方式
Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
for (Schedule s : schedules) {
    System.out.println(s.fixedDelay());
}

// JDK 8 之前的兼容方式
Schedules schedules = method.getAnnotation(Schedules.class);
```

## Spring 中的实际应用

### @PropertySource 多配置

```java
@Configuration
@PropertySource("classpath:config.properties")
@PropertySource("classpath:override.properties")
public class AppConfig { }
```

### JUnit 5 @Tag

```java
@Tag("slow")
@Tag("integration")
@Test
void integrationTest() { }
```

## @Repeatable vs 别名模式

| 特性 | @Repeatable | 别名模式 |
|------|------------|----------|
| 语法 | 多个独立注解 | 数组形式 |
| 可读性 | 更高 | 稍低 |
| JDK 版本 | 8+ | 任意版本 |

