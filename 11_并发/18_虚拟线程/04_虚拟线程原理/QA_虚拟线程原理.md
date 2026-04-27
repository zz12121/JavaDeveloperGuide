
# 虚拟线程原理

## Q1：虚拟线程的底层原理是什么？

**A**：

虚拟线程基于**M:N 调度模型** + **Continuation（续体）**：

1. 大量虚拟线程（M）映射到少量载体线程（N = CPU 核数）
2. 载体线程是 `ForkJoinPool` 中的平台线程
3. 遇到阻塞操作时，虚拟线程的**栈帧保存到 Continuation 中**，与载体线程解绑
4. 阻塞完成后，Continuation 恢复，挂载到其他载体线程继续执行

本质是**用户态协程**，JVM 负责保存/恢复执行上下文。

---

## Q2：什么是 Continuation？

**A**：

Continuation（续体）是虚拟线程的核心组件，代表一个**可暂停和恢复的执行单元**。

```java
Continuation cont = new Continuation(SCOPE, () -> {
    step1();
    Continuation.yield(); // 暂停，保存栈帧
    step2();              // 恢复后从这里继续
});

cont.run();  // 执行到 yield
cont.run();  // 从 yield 处恢复
```

类似 Python 的 generator / Kotlin 的 suspend function。

---

## Q3：什么是 Pinning 问题？

**A**：

Pinning（钉住）是指虚拟线程在 `synchronized` 块或 `native` 方法中执行阻塞操作时，无法释放载体线程，载体线程被"钉住"无法服务其他虚拟线程。

```java
// 可能 pinning
synchronized (lock) {
    socket.read(); // 载体线程被阻塞
}

// 解决方案：用 ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    socket.read(); // 虚拟线程正常挂起
} finally {
    lock.unlock();
}
```

JDK 21+ 对大部分 `synchronized` 场景已优化，但 `native` 方法中的阻塞仍可能导致 pinning。可通过 `-Djdk.tracePinnedThreads=short` 检测 pinning。

