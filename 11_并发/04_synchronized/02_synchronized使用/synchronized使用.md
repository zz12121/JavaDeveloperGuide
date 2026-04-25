---
title: synchronized使用
tags:
  - Java/并发
  - 场景型
module: 04_synchronized
created: 2026-04-18
---

# synchronized使用（修饰实例方法/静态方法/代码块）

## 先说结论

synchronized 有三种使用方式：修饰**实例方法**（锁 this）、修饰**静态方法**（锁 Class 对象）、修饰**代码块**（锁指定对象）。三种方式本质都是获取某个对象的 monitor，区别在于锁的对象不同，作用范围不同。

## 深度解析

### 三种使用方式

```java
// 1. 修饰实例方法：锁当前对象 this
public synchronized void method() { ... }

// 2. 修饰静态方法：锁当前类的 Class 对象
public static synchronized void method() { ... }

// 3. 修饰代码块：锁指定对象
synchronized (obj) { ... }
```

### 锁的范围对比

| 方式       | 锁对象       | 作用范围                     |
| ---------- | ------------ | ---------------------------- |
| 实例方法   | this         | 该实例的所有同步方法互斥     |
| 静态方法   | Class 对象   | 该类的所有同步方法互斥       |
| 代码块     | 指定对象     | 指定对象的同步块互斥         |

### 字节码层面

实例方法和静态方法的 synchronized 在常量池中通过 `ACC_SYNCHRONIZED` 标志实现，代码块通过 `monitorenter/monitorexit` 指令实现。

## 易错点/踩坑

- ❌ 以为不同实例方法的 synchronized 会互斥（实际只有同一实例才互斥）
- ✅ 不同实例之间的 synchronized 方法可以并发执行
- ❌ 在代码块中使用可变对象作为锁（如 String 常量拼接）
- ✅ 锁对象应该是 final 的、不可变的
- ❌ synchronized 作用域过大
- ✅ 尽量缩小同步块范围，只锁必要的临界区

## 代码示例

```java
public class SyncDemo {
    private final Object lock = new Object();
    private static int staticCount = 0;
    private int instanceCount = 0;

    // 实例方法：锁 this
    public synchronized void instanceMethod() {
        instanceCount++;
    }

    // 静态方法：锁 SyncDemo.class
    public static synchronized void staticMethod() {
        staticCount++;
    }

    // 代码块：锁指定对象
    public void blockMethod() {
        // 锁缩小到临界区
        synchronized (lock) {
            instanceCount++;
        }
        // 非临界区代码在锁外面执行
        doSomethingElse();
    }
}
```

## 关联知识点
