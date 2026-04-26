---
title: AtomicBoolean
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicBoolean（compareAndSet实现DCL单例）

## 先说结论

`AtomicBoolean` 提供布尔值的原子操作，最核心的用法是通过 `compareAndSet(expected, newValue)` 实现"只执行一次"的语义，常用于状态标记、DCL 单例模式的锁标志、一次性初始化等场景。

## 深度解析

### 核心方法

```java
public class AtomicBoolean {
    private volatile int value;  // 内部用 int（0=false, 1=true）

    public boolean get();                           // 获取当前值
    public void set(boolean newValue);              // 设置值
    public boolean compareAndSet(boolean expect, boolean update); // CAS
    public boolean getAndSet(boolean newValue);     // 设置并返回旧值
    public boolean weakCompareAndSet(boolean expect, boolean update); // 弱 CAS
}
```

### 典型应用场景

**场景一：状态标记**
```java
AtomicBoolean isRunning = new AtomicBoolean(false);
if (isRunning.compareAndSet(false, true)) {
    // 只有一个线程能进入
    try {
        doWork();
    } finally {
        isRunning.set(false);
    }
}
```

**场景二：DCL 单例的标志位**
```java
public class Singleton {
    private static volatile Singleton instance;
    private static final AtomicBoolean initialized = new AtomicBoolean(false);

    public static Singleton getInstance() {
        if (instance == null) {  // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {  // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**场景三：一次性初始化**
```java
AtomicBoolean initialized = new AtomicBoolean(false);
if (initialized.compareAndSet(false, true)) {
    // 确保初始化只执行一次
    loadConfig();
    initConnectionPool();
}
```

### compareAndSet 的语义

```
compareAndSet(false, true)：
  当前值为 false → 设为 true → 返回 true（成功获取）
  当前值为 true  → 不修改  → 返回 false（已被其他线程获取）

类似于"抢锁"语义：
  false = 未锁定
  true  = 已锁定
```

## 易错点/踩坑

- ❌ 认为 AtomicBoolean 可以替代 synchronized——它只保证单个 CAS 的原子性，不保证代码块的互斥
- ❌ CAS 成功后忘记在 finally 中 reset——可能导致状态永久锁定
- ✅ `compareAndSet` 比加锁轻量，适合简单的"只执行一次"场景

## 代码示例

```java
// 基于 AtomicBoolean 的简单互斥锁
public class SimpleLock {
    private final AtomicBoolean locked = new AtomicBoolean(false);

    public void lock() {
        while (!locked.compareAndSet(false, true)) {
            Thread.yield(); // 自旋等待
        }
    }

    public void unlock() {
        locked.set(false);
    }
}

// 使用
SimpleLock lock = new SimpleLock();
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

## 关联知识点
