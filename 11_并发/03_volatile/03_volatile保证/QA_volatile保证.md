---
title: volatile保证
tags:
  - Java/并发
  - 问答
  - 原理型
module: 03_volatile
created: 2026-04-18
---

# volatile保证（禁止指令重排序，保证可见性）

## Q1：volatile 如何保证有序性（禁止指令重排序）？

**A**：JMM 通过内存屏障来禁止重排序。JMM 制定了 volatile 重排序规则表：

- **volatile 写之前的任何操作**不能重排到 volatile 写之后
- **volatile 读之后的任何操作**不能重排到 volatile 读之前
- **volatile 写不能和 volatile 读**重排序

编译器在生成字节码时会在 volatile 操作前后插入相应的内存屏障指令（LoadLoad/LoadStore/StoreStore/StoreLoad），底层由 CPU 执行 fence 指令来确保。

---

## Q2：volatile 如何保证可见性？

**A**：JMM 规定了 volatile 的 happens-before 规则：**对一个 volatile 变量的写操作，happens-before 于后续对这个变量的读操作**。

具体机制：
- **写操作后**：JMM 会将线程工作内存中的所有共享变量（不仅仅是 volatile 变量）刷新到主内存
- **读操作前**：JMM 会使线程工作内存中的所有共享变量失效，强制从主内存重新加载

这保证了只要线程 A 写入了 volatile 变量，线程 B 一定能读到最新值以及写入之前的所有共享变量的最新值。

---

## Q3：为什么 DCL 单例必须用 volatile？

**A**：`instance = new Singleton()` 在 JVM 层面分三步：
1. 分配内存空间
2. 调用构造方法初始化对象
3. 将引用指向分配的内存

没有 volatile 时，步骤 2 和 3 可能被重排序为 1→3→2。此时线程 B 可能在步骤 3 之后看到 `instance != null`，但对象尚未初始化完成（步骤 2 未执行），导致使用未完全初始化的对象。

volatile 的 StoreStore 屏障确保步骤 2 在步骤 3 之前完成，从而避免这个问题。
---
```java
// volatile 保证有序性 — DCL 单例
class Singleton {
    private static volatile Singleton instance;  // 必须加 volatile
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                  // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {          // 第二次检查（有锁）
                    instance = new Singleton();  // volatile 禁止 1→3→2 重排
                }
            }
        }
        return instance;
    }
}
// 没有 volatile 时，new Singleton() 的步骤可能被重排为：
// 1.分配内存 → 3.instance指向内存 → 2.初始化对象
// 导致其他线程拿到未初始化的对象
```


## 关联知识点
