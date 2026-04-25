---
title: volatile vs synchronized
tags:
  - Java/并发
  - 对比型
module: 03_volatile
created: 2026-04-18
---

# volatile vs synchronized（volatile轻量，synchronized保证原子性）

## 先说结论

volatile 是轻量级同步机制，仅保证可见性和有序性，不保证原子性，不会阻塞线程；synchronized 是互斥锁，同时保证可见性、有序性和原子性，会阻塞线程。两者是**互补关系**而非替代关系。

## 深度解析

### 核心对比

| 维度       | volatile                     | synchronized               |
| ---------- | ---------------------------- | -------------------------- |
| 原子性     | ❌ 不保证                     | ✅ 保证                     |
| 可见性     | ✅ 内存屏障刷新               | ✅ 进入/退出同步块刷新     |
| 有序性     | ✅ 禁止重排序                 | ✅ 互斥即有序               |
| 阻塞       | 不阻塞                       | 会阻塞                     |
| 作用范围   | 修饰变量                     | 修饰方法/代码块            |
| 编译优化   | 禁止部分 JIT 优化            | JIT 可做锁消除/锁粗化      |
| 开销       | 极低（仅内存屏障）           | 较高（可能涉及 OS 上下文切换） |
| 可中断     | N/A                          | 不可中断                   |

### happens-before 关系

- volatile：volatile 写 → 后续 volatile 读
- synchronized：unlock → 后续 lock

### 使用场景

- **volatile**：状态标志、DCL 单例、一写多读配置
- **synchronized**：复合操作、临界区保护、多变量联动

## 易错点/踩坑

- ❌ 认为 volatile 性能一定比 synchronized 好（低竞争下是的，高竞争下 synchronized 优化后可能更好）
- ✅ JDK 6 之后 synchronized 有锁升级，低竞争下开销很小
- ❌ 在计数场景用 volatile 代替 synchronized
- ✅ 计数用 `AtomicInteger`/`LongAdder`

## 代码示例

```java
// volatile：适合状态标志
public class VolatileFlag {
    private volatile boolean running = true;
    public void stop() { running = false; }
    public void work() { while (running) { /* ... */ } }
}

// synchronized：适合复合操作
public class SafeCounter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}
```

## 关联知识点

