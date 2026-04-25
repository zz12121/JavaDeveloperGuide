---
title: volatile写读流程
tags:
  - Java/并发
  - 问答
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile写读流程（写后强制刷新主存，读前invalidate CPU缓存）

## Q1：volatile 写操作的完整流程是什么？

**A**：volatile 写操作在 JIT 编译后的指令序列中插入了两个内存屏障：

1. 在 volatile 写之前插入 **StoreStore 屏障**：确保之前的普通写操作都已经刷新到主内存，不会被重排到 volatile 写之后
2. 执行 volatile 写（写入主内存）
3. 在 volatile 写之后插入 **StoreLoad 屏障**：确保 volatile 写对后续的读操作可见，防止后面的读操作被重排到写之前

其中 **StoreLoad** 是开销最大的屏障，它同时兼具其他三种屏障的功能，几乎等价于一个完整的内存 fence。

---

## Q2：volatile 读操作的完整流程是什么？

**A**：volatile 读操作同样插入两个内存屏障：
1. 在 volatile 读之前插入 **LoadLoad 屏障**：确保之前的读操作已经完成，不会被重排到 volatile 读之后
2. 执行 volatile 读（从主内存重新加载，invalidate 本地缓存行）
3. 在 volatile 读之后插入 **LoadStore 屏障**：确保后续的写操作不会被重排到 volatile 读之前

这些屏障保证了 volatile 读操作一定是在所有之前的操作之后、所有之后的操作之前执行。

---

## Q3：volatile 读写是如何与 MESI 协议配合的？

**A**：
- **写操作**：CPU 执行 volatile 写时，通过总线发送 **Invalidate** 消息，其他 CPU 收到后将本地缓存中该变量的缓存行标记为 **Invalid** 状态，然后当前 CPU 将新值写入主内存
- **读操作**：CPU 检查本地缓存行状态，如果是 **Invalid**，则通过总线从主内存重新加载最新值

volatile 的内存屏障最终是通过 CPU 总线协议（x86 使用 MESI/MESIF）来实现的，JMM 的 volatile 语义是对底层硬件协议的抽象。


> **代码示例：volatile 写-读时序验证**

```java
volatile boolean flag = false;

// 线程A：volatile 写
void writer() {
    data = 42;         // 普通写（在 volatile 写之前，不会被重排到后面）
    flag = true;       // volatile 写（StoreStore + StoreLoad 屏障）
}

// 线程B：volatile 读
void reader() {
    if (flag) {        // volatile 读（LoadLoad + LoadStore 屏障）
        System.out.println(data); // 保证看到 42
    }
}
// volatile 写前的普通写对 volatile 读后的操作可见
```

## 关联知识点

