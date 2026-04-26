---
title: System类常用方法
tags:
  - Java/其他核心概念
  - 原理型
  - 问答
module: 12_其他核心概念
created: 2026-04-18
---

# System类常用方法
## Q1：System.arraycopy 和 Arrays.copyOf 有什么区别？

**A**：
- **System.arraycopy**：需要目标数组已存在并指定位置，可指定源和目标的起始偏移量
- **Arrays.copyOf**：内部调用 arraycopy，但更简洁——自动创建目标数组，支持截断或扩展
```java
int[] src = {1, 2, 3, 4, 5};

// System.arraycopy：目标数组必须存在
int[] dest = new int[5];
System.arraycopy(src, 0, dest, 0, 3); // dest = [1, 2, 3, 0, 0]

// Arrays.copyOf：自动创建目标数组
int[] copy = Arrays.copyOf(src, 3);  // [1, 2, 3]
int[] ext = Arrays.copyOf(src, 7);   // [1, 2, 3, 4, 5, 0, 0]
```
> 性能上几乎相同，Arrays.copyOf 更方便但灵活性不如 arraycopy。

---

## Q2：System.exit 和 return 在 main 方法中的区别？

**A**：
- **return**：仅退出当前方法，正常返回调用者
- **System.exit(int status)**：终止 JVM 虚拟机本身
  - `status = 0` 表示正常退出，非 0 表示异常退出
```java
public static void main(String[] args) {
    try {
        // 业务逻辑
    } finally {
        System.exit(0); // 强制终止 JVM finally 块仍会执行
    }
}
```

---

## Q3：System.currentTimeMillis 和 System.nanoTime 的区别？

**A**：

| 维度 | currentTimeMillis | nanoTime |
|------|-------------------|----------|
| **精度** | 毫秒 | 纳秒（但实际精度依赖系统） |
| **来源** | 系统时钟（可被调整） | 虚拟机单调递增计数器 |
| **用途** | 获取当前时间、与日期互转 | **测量时间间隔、性能基准** |
| **线程安全** | 否 | 是 |

```java
// currentTimeMillis：不适合测量精确时间间隔
long start = System.currentTimeMillis();
Thread.sleep(100); // 操作系统调度可能延迟
long elapsed = System.currentTimeMillis() - start; // 可能 > 100ms

// nanoTime：适合测量精确时间间隔
long start = System.nanoTime();
Thread.sleep(100);
long elapsed = System.nanoTime() - start; // 接近 100ms
```

> **最佳实践**：测量代码执行时间用 `nanoTime`，获取当前时间用 `currentTimeMillis` 或 `Instant.now()`。

---

## Q4：System.gc() 能保证垃圾回收吗？

**A**：
**不能保证**。`System.gc()` 只是**建议**JVM进行垃圾回收，JVM可以忽略这个请求。

```java
System.gc(); // 只是一个提示（Hint），不保证立即回收
System.runFinalization(); // 同理，提示执行 finalize 方法
```

> 面试加分点：在实际项目中，**不要依赖 System.gc()** 来做内存管理，它主要用于调试和测试。正确做法是通过合理的对象作用域和引用管理来让垃圾自动回收。
