---
title: 反射创建对象
tags:
  - Java/反射
  - 场景型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射创建对象

## Q：反射创建对象有哪几种方式？推荐哪种？

**A：** 两种方式：

1. **`Class.newInstance()`**（已弃用）：只能调用 public 无参构造器，异常处理不精确
   ```java
   User user = User.class.newInstance(); // 已弃用
   ```
2. **`Constructor.newInstance()`**（推荐）：可以调用任意构造器（含 private），异常处理更精确
   ```java
   User user = User.class.getDeclaredConstructor(String.class, int.class)
       .newInstance("张三", 25);
   ```
推荐使用 `Constructor.newInstance()`。

## Q：如何通过反射调用私有构造器？

**A：** 使用 `getDeclaredConstructor()` 获取私有构造器，然后调用 `setAccessible(true)` 关闭权限检查：
```java
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);  // 突破 private 限制
Singleton instance = c.newInstance();
```
这也是反射可以**破坏单例模式**的原因。

## Q：setAccessible(true) 的原理是什么？

**A：** `setAccessible(true)` 关闭了 Java 的访问权限检查（Access Control）。调用后，反射可以访问 private 字段、方法和构造器，不再抛出 `IllegalAccessException`。底层通过修改 `AccessibleObject` 对象的 `override` 标志位实现。如果设置了 `SecurityManager`，仍可能被拒绝。
