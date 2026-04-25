---
title: synchronized使用
tags:
  - Java/并发
  - 问答
  - 场景型
module: 04_synchronized
created: 2026-04-18
---

# synchronized使用（修饰实例方法/静态方法/代码块）

## Q1：synchronized 有几种使用方式？

**A**：三种使用方式：

1. **修饰实例方法**：锁当前对象 `this`，同一实例的所有 synchronized 方法互斥
2. **修饰静态方法**：锁当前类的 `Class` 对象，该类所有实例的 synchronized 静态方法互斥
3. **修饰代码块**：锁指定对象，`synchronized(obj)` 中指定任意对象作为锁

---

## Q2：不同使用方式的锁对象分别是什么？

**A**：

- **实例方法** `synchronized void foo()`：锁对象是 `this`
- **静态方法** `static synchronized void foo()`：锁对象是 `Foo.class`
- **代码块** `synchronized(lock)`：锁对象是 `lock`

关键区别：实例方法锁的是**具体实例**，不同实例之间不互斥；静态方法锁的是**Class 对象**，所有实例之间互斥。

```java
SyncDemo a = new SyncDemo();
SyncDemo b = new SyncDemo();
// a.instanceMethod() 和 b.instanceMethod() 可以并发执行
// SyncDemo.staticMethod() 全局只能一个线程执行
```

---

## Q3：synchronized 代码块和 synchronized 方法怎么选？

**A**：优先使用 **synchronized 代码块**，原因：

1. **缩小锁粒度**：只锁必要的临界区代码，提高并发度
2. **灵活选择锁对象**：可以选择更细粒度的锁对象
3. **避免不必要的同步**：非临界区代码在锁外执行，不阻塞其他线程

## 关联知识点