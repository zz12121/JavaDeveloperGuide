---
title: synchronized锁对象
tags:
  - Java/并发
  - 问答
  - 场景型
module: 04_synchronized
created: 2026-04-18
---

# synchronized锁对象（实例方法锁this，静态方法锁Class对象）

## Q1：synchronized 实例方法和静态方法锁的对象分别是什么？

**A**：

- **实例方法** `synchronized void method()`：锁的是**当前实例对象 this**
- **静态方法** `static synchronized void method()`：锁的是**当前类的 Class 对象**（如 `MyClass.class`）

所以两个不同实例调用 synchronized 实例方法不会互斥，但调用 synchronized 静态方法会互斥。静态方法 synchronized 等价于 `synchronized(MyClass.class)`。

---

## Q2：为什么不能用 String 常量作为锁对象？

**A**：Java 的 String 常量会被放入**字符串常量池**，不同代码位置使用相同的字符串字面量实际上指向同一个对象。如果两个不相关的类都使用 `synchronized("LOCK")` 作为锁，它们会互相阻塞。

```java
// 类A和类B中的 "LOCK" 是同一个对象！
class A { void foo() { synchronized ("LOCK") { ... } } }
class B { void bar() { synchronized ("LOCK") { ... } } }
// A.foo() 和 B.bar() 会互相阻塞
```

推荐做法：使用 `private final Object` 作为专用锁对象。

---

## Q3：锁对象可以改变吗？会发生什么？

**A**：如果锁对象被重新赋值，synchronized 锁就失效了，因为 synchronized 锁的是对象实例，不是引用。

```java
private Object lock = new Object();
synchronized (lock) {
    lock = new Object();  // ❌ 危险！其他线程使用的是旧对象
}
```

正确做法是声明为 `final`，确保锁对象不可变。

## 关联知识点

