---
title: 锁消除
tags:
  - Java/并发
  - 问答
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# 锁消除（JIT逃逸分析，无逃逸对象的锁被消除）

## Q1：什么是锁消除？它是如何工作的？

**A**：锁消除是 JIT 编译器的优化手段，基于**逃逸分析**。如果 JIT 编译器通过逃逸分析证明某个对象**不会逃逸出当前线程**（即不可能被其他线程访问），那么该对象上的所有 synchronized 操作都是多余的，JIT 会直接消除这些锁操作。

例如 `StringBuffer` 的 `append()` 方法有 synchronized，但如果 StringBuffer 对象只在方法内使用，JIT 可以安全消除锁。

---

## Q2：逃逸分析有哪几种情况？

**A**：

1. **不逃逸**：对象的作用域完全在方法内部，不会作为返回值、参数传递或赋值给成员变量 → 可以锁消除、标量替换
2. **方法逃逸**：对象被作为参数传递给其他方法，但没有被其他线程访问 → 不可锁消除
3. **线程逃逸**：对象被赋值给静态变量、成员变量，可能被其他线程访问 → 不可锁消除

只有对象**不逃逸**时才能进行锁消除。

---

## Q3：如何确认锁消除是否生效？

**A**：通过 JIT 编译日志确认：

```bash
# 查看编译日志
-XX:+PrintCompilation
# 查看逃逸分析
-XX:+PrintEscapeAnalysis
# 查看锁消除
-XX:+EliminateLocks（默认开启）
-XX:-EliminateLocks（禁用）
```

对比开启和禁用锁消除的性能差异，如果差异显著则说明锁消除生效了。

---

## Q4：StringBuffer vs StringBuilder 在 synchronized 中的区别？

**A**：

| 类 | 线程安全 | 底层实现 | 适用场景 |
|----|----------|----------|----------|
| StringBuilder | ❌ 不安全 | 无 synchronized | 单线程字符串拼接（性能优） |
| StringBuffer | ✅ 安全 | 方法加 synchronized | 多线程共享字符串（安全） |

```java
// StringBuffer 的 append 方法（线程安全但性能差）
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}

// StringBuilder 的 append 方法（无锁，高性能）
public StringBuilder append(String str) {
    super.append(str);
    return this;
}
```

**JIT 优化视角**：如果在单线程中创建 StringBuffer 并使用，JIT 会进行锁消除，效果等同于 StringBuilder。

```java
// JIT 锁消除：单线程环境下，sb 上的锁被消除
StringBuffer sb = new StringBuffer();
sb.append("hello");   // JIT 可能消除 synchronized
sb.append("world");
```

**性能测试对比**（循环拼接 10 万次）：

```
StringBuilder:   ~5ms   （无锁，最快）
StringBuffer:    ~45ms  （有锁，但 JIT 可能锁消除）
+ 手动 synchronized: ~200ms （显式锁开销最大）
```

**选择建议**：
- 明确单线程 → StringBuilder
- 明确多线程共享 → StringBuffer 或手动加锁
- 不确定时 → 优先 StringBuilder，线程安全交给外部保证

---

## Q5：逃逸分析除了锁消除还有什么优化？

**A**：

逃逸分析（JVM 通过 JIT 编译器分析对象动态作用域）能带来三大优化：

| 优化 | 原理 | 效果 |
|------|------|------|
| **锁消除** | 对象不逃逸，消除 synchronized | 减少同步开销 |
| **栈分配** | 对象在栈上分配而非堆 | 减少 GC 压力 |
| **标量替换** | 对象拆解为原始类型，直接在寄存器/栈上操作 | 避免对象分配 |

```java
// 逃逸分析开启时（默认）
public void process() {
    Point p = new Point(1, 2);  // JIT 分析：p 不逃逸
    // 直接使用 p.x, p.y，不实际分配对象
    System.out.println(p.x + p.y);
}

// 禁用逃逸分析：-XX:-DoEscapeAnalysis
// 对象会被分配到堆上，触发 GC
```

**验证逃逸分析**：
```bash
-XX:+PrintEscapeAnalysis  # 打印逃逸分析结果
-XX:+PrintGCDetails       # 观察是否有栈分配（Allocation=Stack）
```

> **注意**：逃逸分析有开销，JIT 会分析热点代码后决定是否优化。解释执行阶段没有这些优化。

---

## 关联知识点
