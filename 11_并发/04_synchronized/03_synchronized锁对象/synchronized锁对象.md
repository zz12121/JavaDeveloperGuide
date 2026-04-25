---
title: synchronized锁对象
tags:
  - Java/并发
  - 场景型
module: 04_synchronized
created: 2026-04-18
---

# synchronized锁对象（实例方法锁this，静态方法锁Class对象）

## 先说结论

synchronized 的本质是获取某个**对象的 monitor（监视器锁）**。实例方法锁 this，静态方法锁 Class 对象，代码块锁指定对象。锁对象的选择直接决定了同步的作用范围——不同锁对象之间不会互斥。

## 深度解析

### 锁对象与作用范围

| 使用方式     | 锁对象           | 互斥范围                   |
| ------------ | ---------------- | -------------------------- |
| synchronized 实例方法 | this             | 同一实例的同步方法之间     |
| synchronized 静态方法 | Xxx.class        | 该类所有实例的同步静态方法 |
| synchronized(obj)     | obj              | 同一 obj 的同步块之间     |

### 常见锁对象选择

```java
// 1. 锁 this：保护实例状态
public synchronized void addCount() { this.count++; }

// 2. 锁 Class：保护类级别状态（如静态变量）
public static synchronized void addGlobalCount() { globalCount++; }

// 3. 锁专用对象：细粒度控制
private final Object lock = new Object();
public void doSomething() {
    synchronized (lock) { /* 临界区 */ }
}

// 4. 锁 String 常量池：危险！
synchronized ("lock") { ... }  // ❌ 可能和其他类的 "lock" 冲突

// 5. 锁枚举值：安全且语义清晰
synchronized (LockEnum.MY_LOCK) { ... }  // ✅ 推荐
```

### 锁对象的本质

Java 中每个对象都有一个 monitor，存储在对象头的 Mark Word 中。synchronized 就是在进入同步块时获取 monitor，退出时释放 monitor。

## 易错点/踩坑

- ❌ 用 String 常量或包装类作为锁对象（常量池导致锁共享）
- ✅ 用 `private final Object` 作为专用锁对象
- ❌ 锁对象用可变对象（重新赋值后锁失效）
- ✅ 锁对象应该是 final 的
- ❌ 混淆 this 锁和 Class 锁的互斥范围
- ✅ 实例方法只锁实例，静态方法锁整个类

## 代码示例

```java
public class LockObjectDemo {
    private int a = 0;
    private int b = 0;
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    // 操作 a 用 lockA，操作 b 用 lockB，两者不互斥
    public void incrementA() {
        synchronized (lockA) { a++; }
    }

    public void incrementB() {
        synchronized (lockB) { b++; }
    }

    // 同时需要 a 和 b 时，按固定顺序获取两把锁避免死锁
    public void incrementBoth() {
        synchronized (lockA) {
            synchronized (lockB) {
                a++; b++;
            }
        }
    }
}
```

## 关联知识点

