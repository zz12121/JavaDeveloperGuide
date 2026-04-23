---
title: 反射访问字段
tags:
  - Java/反射
  - 场景型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射访问字段

## Q：如何通过反射读写对象的字段？

**A：** 通过 `Field.get()` 读取、`Field.set()` 修改：

```java
// 获取字段
Field field = clazz.getDeclaredField("fieldName");
field.setAccessible(true);  // private 字段需要

// 读取
Object value = field.get(target);

// 修改
field.set(target, newValue);
```

static 字段访问时对象参数传 null：`field.get(null)`、`field.set(null, value)`。

## Q：如何通过反射修改 final 字段？

**A：** 需要先去除 `final` 修饰符，再修改值：
```java
Field field = clazz.getDeclaredField("CONSTANT");
field.setAccessible(true);

// 去除 final 修饰符
Field modifiers = Field.class.getDeclaredField("modifiers");
modifiers.setAccessible(true);
modifiers.setInt(field, field.getModifiers() & ~Modifier.FINAL);

field.set(null, newValue);
```
注意：`static final` 基本类型常量如果被编译器内联，其他类看到的仍是旧值。

## Q：Field 提供了哪些基本类型的专用方法？

**A：** `Field` 为每种基本类型提供了 `getXxx()`/`setXxx()` 方法，避免装箱拆箱：
- `getInt()` / `setInt()`
- `getLong()` / `setLong()`
- `getBoolean()` / `setBoolean()`
- `getDouble()` / `setDouble()`
也可以用通用方法 `get()` / `set()`，基本类型会自动装箱/拆箱。
