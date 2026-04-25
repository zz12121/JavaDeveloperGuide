---
title: Happens-Before规则
tags:
  - Java/并发
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# Happens-Before 规则（程序顺序/锁/volatile/线程启动/中断/终结/传递性）

## 先说结论

Happens-Before（先行发生原则）是JMM中定义的两个操作之间的**偏序关系**：如果操作A Happens-Before 操作B，那么A的结果对B可见，且A在B之前执行。JMM定义了8种天然的Happens-Before规则：程序顺序、监视器锁、`volatile`、线程启动、线程终止、中断、终结器、`传递性`。这是判断多线程程序可见性和有序性的核心依据。

## 深度解析

### 1. 什么是Happens-Before

**定义**：
- 如果操作A Happens-Before 操作B（记作 A → B）
- 则A的结果对B**可见**
- 且A在B之前**执行**（禁止重排序）

**核心作用**：
- 解决多线程环境下的**可见性**问题
- 解决编译器和处理器的**重排序**问题
- 为程序员提供简单的并发编程规则

### 2. 8种Happens-Before规则详解

#### 2.1 程序顺序规则（Program Order Rule）

```
在一个线程内，按照程序代码顺序，前面的操作Happens-Before后面的操作
```

**注意**：只保证单线程内的顺序，多线程之间不保证。

#### 2.2 监视器锁规则（Monitor Lock Rule）

```
对一个锁的unlock操作Happens-Before后续对这个锁的lock操作
```

```java
synchronized (obj) {
    x = 1;  // 操作A
}           // unlock

synchronized (obj) {
    // 操作B能看到 x = 1
}
```

#### 2.3 volatile变量规则（Volatile Variable Rule）

```
对volatile变量的写操作Happens-Before后续对这个变量的读操作
```

```java
volatile int x;

x = 1;      // 写操作
// ...
int y = x;  // 读操作，保证看到 x = 1
```

#### 2.4 线程启动规则（Thread Start Rule）

```
Thread对象的start()方法Happens-Before此线程的每一个动作
```

```java
x = 1;                  // 操作A
thread.start();         // 启动线程
// 线程内能看到 x = 1
```

#### 2.5 线程终止规则（Thread Termination Rule）

```
线程中的所有操作都Happens-Before对此线程的终止检测
```

```java
thread.start();
thread.join();          // 等待线程结束
// join返回后，线程中的所有操作结果都可见
```

#### 2.6 中断规则（Interruption Rule）

```
对线程interrupt()的调用Happens-Before被中断线程检测到中断事件
```

```java
thread.interrupt();     // 中断线程
// 线程内调用 isInterrupted() 或捕获 InterruptedException 时可见
```

#### 2.7 终结器规则（Finalizer Rule）

```
对象的构造函数执行Happens-Before它的finalize()方法
```

```java
public class Resource {
    private Object data;
    
    public Resource() {
        this.data = new Object();  // 构造函数
    }
    
    @Override
    protected void finalize() {
        // 保证能看到构造函数中对data的赋值
        cleanup(data);
    }
}
```

#### 2.8 传递性规则（Transitivity）

```
如果 A → B，且 B → C，那么 A → C
```

```java
volatile int x;
int y;

// 线程A
y = 1;      // 操作A（程序顺序）
x = 2;      // 操作B（volatile写）

// 线程B
int r1 = x; // 操作C（volatile读）
int r2 = y; // 操作D
// 由于 A→B（程序顺序），B→C（volatile规则），根据传递性 A→C→D
// 所以 r2 保证看到 y = 1
```

### 3. Happens-Before vs as-if-serial

| 特性 | Happens-Before | as-if-serial |
|------|---------------|--------------|
| 适用范围 | 多线程 | 单线程 |
| 保证内容 | 可见性 + 有序性 | 执行结果正确性 |
| 约束强度 | 禁止特定重排序 | 允许不影响结果的重排序 |
| 关系 | 多线程正确性的基础 | 单线程优化的基础 |

## 易错点/踩坑

- ❌ **错误理解**：Happens-Before表示时间上的先后
  - ✅ **正确**：Happens-Before是**语义上的先后**，实际执行可能重叠

- ❌ **错误理解**：A Happens-Before B 意味着A先执行完B才开始
  - ✅ **正确**：只保证A的结果对B可见，执行可以重叠

- ❌ **错误理解**：所有操作都有Happens-Before关系
  - ✅ **正确**：只有满足8种规则的操作之间才有

- ❌ **错误理解**：`volatile`能保证复合操作的原子性
  - ✅ **正确**：`volatile`只保证单次读写的可见性，`i++`仍需同步

- ❌ **错误理解**：线程join返回后所有共享变量都可见
  - ✅ **正确**：只保证被join线程中的操作可见，不保证其他线程

## 代码示例

```java
public class HappensBeforeDemo {
    private int a = 0;
    private volatile int b = 0;
    
    public void writer() {
        a = 1;          // 操作1
        b = 2;          // 操作2（volatile写）
    }
    
    public void reader() {
        int r1 = b;     // 操作3（volatile读）
        int r2 = a;     // 操作4
        
        // 如果 r1 == 2，则保证 r2 == 1
        // 因为：操作1 → 操作2（程序顺序）
        //       操作2 → 操作3（volatile规则）
        //       操作3 → 操作4（程序顺序）
        //       根据传递性：操作1 → 操作4
    }
}
```

## 图解/流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Happens-Before 规则图解                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   单线程内：                                                         │
│   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐                         │
│   │ op1 │───►│ op2 │───►│ op3 │───►│ op4 │  （程序顺序规则）         │
│   └─────┘    └─────┘    └─────┘    └─────┘                         │
│                                                                     │
│   锁规则：                                                           │
│   线程A:              线程B:                                         │
│   ┌────────┐         ┌────────┐                                    │
│   │ unlock │────────►│  lock  │  （监视器锁规则）                    │
│   └────────┘         └────────┘                                    │
│                                                                     │
│   volatile规则：                                                     │
│   线程A:              线程B:                                         │
│   ┌────────┐         ┌────────┐                                    │
│   │ write  │────────►│  read  │  （volatile变量规则）                │
│   └────────┘         └────────┘                                    │
│                                                                     │
│   传递性示例：                                                        │
│   ┌─────┐    ┌────────┐    ┌────────┐    ┌─────┐                   │
│   │ a=1 │───►│ b=2(v) │───►│ read b │───►│read a│                  │
│   └─────┘    └────────┘    └────────┘    └─────┘                   │
│     │            │              │            │                      │
│     └────────────┴──────────────┴────────────┘                      │
│              通过传递性保证可见性                                     │
└─────────────────────────────────────────────────────────────────────┘
```


### Happens-Before 规则关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    八大 Happens-Before 规则                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 程序次序规则      同一线程内，按代码顺序                     │
│         │                                                       │
│  2. 管程锁定规则      unlock → lock（同一锁）                    │
│         │                                                       │
│  3. volatile规则      写 → 读（同一变量）                        │
│         │                                                       │
│  4. 线程启动规则      start() → 线程内操作                       │
│         │                                                       │
│  5. 线程终止规则      线程操作 → join()返回                      │
│         │                                                       │
│  6. 线程中断规则      interrupt() → 检测中断                    │
│         │                                                       │
│  7. 对象终结规则      构造函数 → finalize()                      │
│         │                                                       │
│  8. 传递性规则        A→B，B→C ⇒ A→C                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

分析并发问题的步骤：
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 1.找出   │ → │ 2.匹配   │ → │ 3.判断   │ → │ 4.得出   │
│   所有操作│    │   规则    │    │   关系    │    │   结论    │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### `final` 字段的 Happens-Before 规则

**规则内容**：

> 对象构造函数的完成 **Happens-Before** 该对象 final 字段的任意后续读取操作。

**含义**：只要对象正确发布，其他线程在读取到对象引用后，**无需同步就能看到** final 字段的正确值，而不仅仅是默认值。

```java
public class FinalFieldHB {
    final int x;
    int y;  // 非 final

    public FinalFieldHB() {
        x = 1;   // final 字段写入
        y = 2;   // 普通字段写入
    }
}

// 线程A
FinalFieldHB obj = new FinalFieldHB();  // 构造完成

// 线程B
if (obj != null) {
    System.out.println(obj.x);  // ✅ 一定看到 x = 1（Happens-Before 保证）
    System.out.println(obj.y);  // ❌ 可能看到 y = 0（无保证）
}
```

**前提条件**（避免 this 逃逸）：
- 不要在构造函数中 `return this`
- 不要在构造函数中启动线程
- 不要在构造函数中将 `this` 注册到其他线程可见的集合中

### 并发工具类的隐式 Happens-Before

JUC 中的同步工具类提供了隐式的 Happens-Before 保证，是面试高频考点：

```java
// 1. CountDownLatch：countDown() → await() 有 Happens-Before
CountDownLatch latch = new CountDownLatch(1);
latch.countDown();  // 操作A
latch.await();      // 操作B：一定能看到 countDown() 之前的所有操作

// 2. CyclicBarrier：线程在 barrier 处的操作有 Happens-Before
// barrier.await() 之前的操作对其他线程可见

// 3. Semaphore：release() → acquire() 有 Happens-Before

// 4. FutureTask：run() 内部操作 → get() 有 Happens-Before
FutureTask<String> task = new FutureTask<>(() -> "result");
new Thread(task).start();
String result = task.get();  // 一定能看到线程中的所有操作
```

**规律**：所有 JUC 同步工具的 `await()/acquire()/get()` 方法之前的所有操作，都对执行完成的线程有 Happens-Before 保证。

## 关联知识点

