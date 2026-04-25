---
title: volatile应用场景
tags:
  - Java/并发
  - 场景型
module: 03_volatile
created: 2026-04-18
---

# volatile应用场景（状态标志/双重检查锁/单例模式）

## 先说结论

volatile 的核心适用场景是**一写多读的状态标志位**和**DCL 双重检查锁单例模式**。关键判断标准：**变量是否独立于其他变量**（不参与复合操作），如果是，volatile 就够用；如果涉及多个变量的原子更新，则需要 synchronized 或 Lock。

## 深度解析

### 场景一：状态标志位（最经典）

```java
private volatile boolean shutdownRequested;

public void shutdown() {
    shutdownRequested = true;
}

public void doWork() {
    while (!shutdownRequested) {
        // 执行业务逻辑
    }
}
```

适用条件：只有一个线程写，多个线程读，不涉及复合操作。

### 场景二：DCL 双重检查锁单例

```java
private static volatile Singleton instance;
```

防止 `new Singleton()` 中的指令重排序，确保其他线程看到的对象一定是完全初始化过的。

### 场景三：开销较低的"读-写"模式

当读远多于写时，synchronized 的互斥开销太大，volatile 可以提供无锁的可见性保证。

### 场景四：Calendar 修复

```java
// JDK 源码中 Calendar 类的修复
private volatile boolean calendarIsInitialized;
private volatile Calendar cachedCalendar;
```

使用双 volatile 保证延迟初始化的可见性。

### 不适用场景

- 计数器 `i++`：复合操作，不保证原子性
- 多变量联动更新：如同时更新 x 和 y
- 复杂的条件判断+操作：如 check-then-act

## 易错点/踩坑

- ❌ 在高并发计数场景用 volatile 代替 AtomicLong
- ✅ 计数场景用 `AtomicInteger`/`LongAdder`
- ❌ 用 volatile 修饰 final 字段（final 本身保证可见性）
- ✅ volatile 只修饰可变的共享变量
- ❌ 认为 volatile 单例就能完全替代枚举单例
- ✅ 枚举单例仍然是最佳实践

## 代码示例

```java
// 场景：多线程任务调度器
public class TaskScheduler {
    private volatile boolean running = true;
    private volatile long lastUpdateTime = 0;

    public void stop() {
        running = false;
    }

    public void updateSchedule() {
        lastUpdateTime = System.currentTimeMillis();
    }

    public void run() {
        while (running) {
            if (lastUpdateTime > 0) {
                long elapsed = System.currentTimeMillis() - lastUpdateTime;
                if (elapsed > 5000) {
                    // 超时处理
                    System.out.println("调度超时: " + elapsed + "ms");
                }
            }
        }
    }
}
```

## 关联知识点
