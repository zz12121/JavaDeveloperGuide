---
title: 线程中断
tags:
  - Java/并发
  - 问答
  - 场景型
module: 01_线程基础
created: 2026-04-18
---

# 线程中断（interrupt() / isInterrupted() / interrupted()，设置中断标志）

## Q1：Java 线程中断的本质是什么？

**A**：Java 线程中断是一种**协作式**的通信机制，不是强制终止：
- `interrupt()` 设置目标线程的**中断标志**为 true
- 目标线程通过 `isInterrupted()` 或 `interrupted()` 检测标志
- 如果线程处于阻塞状态（sleep/wait/join），会**立即抛出 InterruptedException**
- 线程收到中断后可以选择优雅退出

---

## Q2：interrupt()、isInterrupted()、interrupted() 三个方法有什么区别？

**A**：

| 方法 | 类型 | 作用 | 是否清除标志 |
|------|------|------|--------------|
| `interrupt()` | 实例方法 | 设置目标线程的中断标志 | 阻塞时清除 |
| `isInterrupted()` | 实例方法 | 判断目标线程是否被中断 | **否** |
| `interrupted()` | 静态方法 | 判断**当前线程**是否被中断 | **是** |

---

## Q3：线程处于阻塞状态时收到中断会怎样？

**A**：当线程处于以下阻塞状态时收到 `interrupt()`：
- `sleep()` → 抛出 `InterruptedException`
- `wait()` → 抛出 `InterruptedException`
- `join()` → 抛出 `InterruptedException`
- `BlockingQueue` 操作 → 抛出 `InterruptedException`

**重要**：这些方法抛出异常时会**同时清除中断标志**！

---

## Q4：正确的响应中断方式？

**A**：
```java
// 推荐方式：在 catch 中重新设置中断标志
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // 重新设置
    return;  // 或 break
}
```

---

## Q5：为什么说 Java 中断是协作式的而不是强制的？

**A**：因为 `interrupt()` 只是设置一个标志位，不会直接终止线程。线程必须主动检查这个标志（通过 `isInterrupted()` 或捕获 `InterruptedException`）才能响应中断。这种设计给予线程机会在收到中断前完成必要的清理工作，保证程序的健壮性。

---

## Q6：InterruptedException 抛出后中断标志会被清除，能恢复吗？

**A**：会。抛出 `InterruptedException` 时，JVM 会清除中断标志。如果需要保留中断状态，可以在 catch 块中调用 `Thread.currentThread().interrupt()` 重新设置中断标志。这是一种推荐的实践。

---

## Q7：interrupted() 和 isInterrupted() 的区别？

**A**：主要区别有两点：
1. `interrupted()` 是静态方法，操作**当前线程**；`isInterrupted()` 是实例方法，操作**目标线程**
2. `interrupted()` 调用后会**清除中断标志**；`isInterrupted()` 不会清除标志

## 关联知识点
