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

#### 字节码详解：monitorenter / monitorexit

```java
// 代码示例
public class SyncBytecode {
    private final Object lock = new Object();

    // 方法A：代码块形式
    public void blockMethod() {
        synchronized (lock) {   // monitorenter
            doSomething();
        }                        // monitorexit
    }

    // 方法B：实例方法形式
    public synchronized void instanceMethod() {
        doSomething();
    }

    // 方法C：静态方法形式
    public static synchronized void staticMethod() {
        doSomething();
    }
}
```

**对应的字节码**：

```java
// 方法A：代码块（显式 monitorenter/monitorexit）
public void blockMethod();
  Code:
     0: aload_1           // 加载 lock 对象引用
     1: dup                // 复制引用（用于 monitorexit）
     2: astore_2          // 存到局部变量槽2
     3: monitorenter       // ====== 进入同步块，获取 monitor ======
     4: invokestatic  #2   // doSomething()
     7: aload_2           // 加载 lock 引用
     8: monitorexit        // ====== 退出同步块，释放 monitor ======
     9: goto          17
    12: astore_3          // 异常处理器：存异常
    13: aload_2
    14: monitorexit        // ====== 异常路径也要释放 monitor ======
    15: aload_3
    16: athrow
    17: return

  Exception table:
     from    to  target  type
         4     9       12   any    // 异常处理：确保 monitor 释放

// 方法B：实例方法（ACC_SYNCHRONIZED 隐式）
public synchronized void instanceMethod();
  flags: ACC_SYNCHRONIZED      // ====== 方法标志位，表示整个方法体加锁 ======
  Code:
     0: invokestatic  #2
     3: return

// 方法C：静态方法（ACC_SYNCHRONIZED 隐式）
public static synchronized void staticMethod();
  flags: ACC_STATIC, ACC_SYNCHRONIZED  // ====== 对 Class 对象加锁 ======
  Code:
     0: invokestatic  #2
     3: return
```

#### 三种形式的本质对比

| 形式 | 字节码机制 | 锁对象 | 临界区 | 异常处理 |
|------|------------|--------|--------|----------|
| **代码块** | `monitorenter/monitorexit` | 显式指定 | 显式 | 需要 try-finally 保证释放 |
| **实例方法** | `ACC_SYNCHRONIZED` | `this` 对象 | 整个方法体 | JVM 自动处理 |
| **静态方法** | `ACC_SYNCHRONIZED` | `Class` 对象 | 整个方法体 | JVM 自动处理 |

#### 关键原理

```java
// monitorenter 的执行步骤（JVM 规范）
1. 如果 monitor 计数为 0 → 进入并置为 1（成功）
2. 如果是同一线程重入 → 计数 +1（可重入）
3. 如果是其他线程持有 → 阻塞等待

// monitorexit 的执行步骤
1. 计数 -1
2. 如果计数归 0 → 唤醒其他等待线程
```

**为什么实例/静态方法不需要 try-finally？**

因为 `ACC_SYNCHRONIZED` 方法由 JVM 解释执行时，JVM 会自动包装为：
```java
monitorenter(this);
try {
    // 方法体
} finally {
    monitorexit(this);
}
```

**面试加分点**：
- `monitorenter` 失败时线程会被阻塞，而 `Lock.tryLock()` 可以立即返回
- `synchronized` 不可中断，而 `ReentrantLock.lockInterruptibly()` 可响应中断
- 字节码中 `monitorexit` 有两条路径（正常返回 + 异常处理），保证锁一定释放

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
