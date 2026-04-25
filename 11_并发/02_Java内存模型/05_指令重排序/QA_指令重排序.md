---
title: 指令重排序
tags:
  - Java/并发
  - 问答
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# 指令重排序（编译器/CPU/HBA重排序三层次，volatile禁止重排序）

## Q1：什么是指令重排序？为什么要进行指令重排序？

**A**：指令重排序是指编译器和处理器为了优化程序性能，在**不改变单线程执行结果**的前提下，调整指令执行顺序的机制。

**重排序的原因：**

1. **充分利用 CPU 流水线**
   - CPU 流水线可以并行执行无依赖的指令
   - 减少流水线停顿（Stall）

2. **隐藏内存访问延迟**
   - CPU 计算速度 >> 内存访问速度（数百倍差距）
   - 在等待内存数据时执行其他指令

3. **提高指令级并行度**
   - 多个独立操作同时执行
   - 充分利用多发射（Multi-Issue）技术

**示例：**
```java
// 原始代码（按顺序执行）
int a = 1;      // 等待内存
int b = 2;      // 等待内存
int c = a + b;   // 等待 a, b 准备好

// 优化后（隐藏延迟）
load b from memory → 计算 b+1 → load a from memory → 计算 a+b
// b 的加载和 a+b 的计算并行执行
```

---

## Q2：指令重排序分为哪几个层次？各有何特点？

**A**：重排序分为三个层次：

| 层次 | 执行主体 | 优化方式 | 可控性 |
|------|---------|---------|--------|
| **编译器重排序** | JIT 编译器 | 调整源码级指令顺序 | 可通过 happens-before 规则禁止 |
| **CPU 指令重排序** | 处理器执行单元 | 流水线乱序执行 | 可通过内存屏障禁止 |
| **内存系统重排序** | Store Buffer、CPU Cache | 写缓冲延迟、缓存一致性 | 可通过内存屏障禁止 |

**详细说明：**

**1. 编译器重排序**
- 最激进的优化
- 在**编译期间**完成
- Java：JIT 编译器在运行时优化
- 必须遵守数据依赖性

**2. CPU 指令重排序**
- 在**执行期间**完成
- 保留性（WAR/WAW/RAW）依赖才阻塞
- 现代 CPU 大多支持乱序执行

**3. 内存系统重排序（HBA）**
- Store Buffer 延迟写回主内存
- Invalidate Queue 延迟处理失效消息
- 导致 CPU 看到的内存顺序与程序顺序不一致

---

## Q3：什么是 as-if-serial 语义？

**A**：as-if-serial（好像串行）是编译器和处理器必须遵守的核心原则：

> **在不改变单线程程序执行结果的前提下，可以进行任意重排序。**

**含义：**
- 单线程程序看到的执行顺序"好像"是严格按照代码顺序执行的
- 即使发生了重排序，单线程的执行结果必须与串行执行完全一致

**约束条件：**
1. **数据依赖性不能破坏**
   ```java
   // 不能重排序
   a = 1;    // 写
   b = a;    // 读后写 - 依赖上面的写
   ```

2. **控制依赖性需要特殊处理**
   ```java
   if (flag) {      // 控制依赖
       a = 1;
   }
   // 编译器可以"猜测执行" a=1，但不能无限优化
   ```

**为什么需要这个语义？**
- 允许编译器/CPU 进行激进优化
- 同时保证单线程程序语义不变
- 多线程问题需要额外的同步机制

---

## Q4：多线程环境下，重排序会导致什么问题？

**A**：重排序在多线程环境下可能导致**可见性**和**有序性**问题。

**问题示例：**
```java
public class ReorderProblem {
    private int a = 0;
    private int b = 0;

    public void writer() {
        a = 1;     // 操作1
        b = 2;     // 操作2（可能被重排序到操作1之前）
    }

    public void reader() {
        int x = b; // 操作3
        int y = a; // 操作4（可能被重排序到操作3之前）
        // x=2 && y=0 的情况
    }
}
```

**线程 B 可能看到：**
- `b=2` 但 `a=0`（a 的赋值被重排序到 b 之后）

**为什么单线程没问题？**
- 程序次序规则保证：操作1 happens-before 操作2
- 编译器不会重排序有依赖的操作
- 但**多线程间没有 happens-before 关系**，所以可能重排序

**解决方案：**
```java
// 方案1：volatile
private volatile int b = 0;

// 方案2：synchronized
public synchronized void writer() {
    a = 1;
    b = 2;
}
```

---

## Q5：volatile 是如何禁止重排序的？

**A**：volatile 通过插入**内存屏障**来禁止特定类型的重排序。

**volatile 的语义：**
1. **可见性**：volatile 变量的读写直接与主内存交互
2. **有序性**：volatile 操作前后的指令不能重排序

**JVM 对 volatile 插入的屏障：**

```java
public void writer() {
    a = 1;           // 普通写
    flag = true;    // volatile 写
    // 插入 StoreStore 屏障：禁止上面的写重排序到 volatile 写之后
    // 插入 StoreLoad 屏障：禁止下面的读写重排序到 volatile 读之前
}

public void reader() {
    // 插入 LoadLoad 屏障：禁止上面的读重排序到 volatile 读之前
    // 插入 LoadStore 屏障：禁止上面的写重排序到 volatile 读之前
    if (flag) {     // volatile 读
        int i = a;  // 普通读 - 一定能读到 a=1
    }
}
```

**屏障类型的作用：**

| 屏障类型 | 作用 |
|---------|------|
| StoreStore | 禁止 **普通写** 重排序到 **volatile 写** 之后 |
| StoreLoad | 禁止 **volatile 写** 之后的 **读写** 重排序 |
| LoadLoad | 禁止 **普通读** 重排序到 **volatile 读** 之后 |
| LoadStore | 禁止 **普通读** 重排序到 **volatile 写** 之前 |

---

## Q6：数据依赖性是什么？哪些情况不能重排序？

**A**：数据依赖性是指**指令之间的依赖关系**，存在数据依赖的操作不能重排序。

**三种数据依赖类型：**

| 类型 | 示例 | 说明 |
|------|------|------|
| **Read-After-Write (RAW)** | `a=1; b=a;` | 读依赖于写 |
| **Write-After-Read (WAR)** | `c=b; c=2;` | 写依赖于读 |
| **Write-After-Write (WAW)** | `a=1; a=2;` | 写依赖于写 |

**示例：**
```java
int a = 1;     // 操作1
int b = 2;     // 操作2
int c = a + b; // 操作3 - 依赖操作1和操作2
```
- 操作1 → 操作3：RAW 依赖（不能重排序）
- 操作2 → 操作3：RAW 依赖（不能重排序）
- 操作1 → 操作2：**可以重排序**（无依赖）

**注意：**
- 数据依赖性是**硬件层面**的概念
- 编译器必须遵守数据依赖性
- CPU 乱序执行也必须遵守数据依赖性

---

## Q7：单例模式中，构造函数指令重排序会导致什么问题？

**A**：这是_double-checked locking_ 模式的经典问题。

**问题代码：**
```java
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**构造函数的重排序：**
`instance = new Singleton()` 分解为：
1. 分配内存
2. 调用构造函数
3. 将引用写入 instance

**可能发生的重排序（1 → 3 → 2）：**
```
时间 →
线程A: [分配内存] → [写入instance] → [调用构造函数]
线程B:                              → 看到instance!=null → 使用不完整对象！
```

**问题后果：**
- 线程 B 获得一个**引用非空但未完全构造**的对象
- 访问未初始化的字段导致 NullPointerException
- 访问部分初始化的数据导致数据不一致

**解决方案：volatile 禁止重排序**
```java
private volatile static Singleton instance;
```
- volatile 插入的 StoreLoad 屏障
- 保证构造函数完全执行后，才能写入 instance

---

## Q8：`synchronized` 是如何阻止重排序的？和 `volatile` 有什么区别？

**A**：

`synchronized` 通过在锁获取和释放操作前后插入内存屏障，禁止临界区内的操作与外部操作之间的重排序：

```java
// synchronized 的屏障序列
synchronized (obj) {
    a = 1;
}
// monitorexit 后插入 StoreStore + StoreLoad 屏障
```

**`synchronized` vs `volatile` 对比**：

| 对比项 | `synchronized` | `volatile` |
|--------|---------------|-----------|
| **原子性** | ✅ 保证（互斥） | ❌ 不保证复合操作 |
| **可见性** | ✅ 释放锁时刷新所有缓存 | ✅ 立即刷新（写屏障） |
| **有序性** | ✅ 禁止临界区内外重排序 | ✅ 禁止与 volatile 相关的重排序 |
| **重排序范围** | 整个 synchronized 块 | 只针对 volatile 变量本身 |
| **性能** | 较重（加锁开销） | 较轻（仅屏障开销） |

```java
// 示例：synchronized 禁止块内外重排序
a = 1;                    // A
synchronized (obj) {      // monitorenter + LoadLoad + LoadStore
    b = 2;               // B（临界区内）
}                        // monitorexit + StoreStore + StoreLoad
c = 3;                   // C

// ✅ A 不会重排序到 B 之后
// ✅ B 不会重排序到 C 之前
// ✅ 这与 volatile 只针对特定变量的语义不同
```

> **面试结论**：`synchronized` 是"范围级"禁止重排序，`volatile` 是"变量级"禁止重排序；`synchronized` 额外保证原子性，`volatile` 不保证。

## 关联知识点