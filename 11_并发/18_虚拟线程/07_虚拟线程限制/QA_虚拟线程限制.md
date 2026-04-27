
# 虚拟线程限制

## Q1：什么是 Pinning？为什么虚拟线程会发生 Pinning？

**A**：

**Pinning（钉住）** 指虚拟线程在执行阻塞操作时，无法从载体线程上卸载（unmount），导致载体线程被占用，无法服务其他虚拟线程。

**根本原因**：`synchronized` 使用 monitor 机制，monitor 的所有权与载体线程绑定。当虚拟线程持有 monitor 并进入阻塞时，JVM 无法安全地将其卸载，因为 monitor 状态依赖于载体线程的栈。

**影响**：在高并发场景下，Pinning 会严重降低吞吐量，因为载体线程数量有限（等于 CPU 核数），每个被钉住的载体线程都是一个浪费。

---

## Q2：如何解决 Pinning 问题？

**A**：

**方案一：替换 synchronized 为 ReentrantLock（推荐）**

```java
// ❌ 可能 pinning
synchronized (lock) {
    blockingIO();
}

// ✅ 使用 ReentrantLock
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingIO();  // 虚拟线程正常挂起，不 pinning
} finally {
    lock.unlock();
}
```

**方案二：升级 JDK 版本**
- JDK 21+ 已通过 JEP 491 优化大部分 `synchronized` 的 pinning
- 仅在 `synchronized` 块内调用 Native 方法阻塞时才会发生

**方案三：检测 Pinning**
```bash
-Djdk.tracePinnedThreads=short   # 简短输出
-Djdk.tracePinnedThreads=full    # 完整堆栈
```

---

## Q3：虚拟线程在 ThreadLocal 方面有什么限制？

**A**：

每个虚拟线程都有自己的 `ThreadLocal` 副本。虚拟线程可以轻松创建百万个，如果每个虚拟线程都存储较大的 ThreadLocal 数据，会导致**严重的内存膨胀**。

例如：每个 ThreadLocal 存 1KB 数据，100 万个虚拟线程 = 1GB 内存。

解决方案是使用 **Scoped Values**（作用域值，JDK 21 Preview / JDK 22+ 正式），它通过不可变的共享值替代 ThreadLocal 的可变线程私有副本，避免了内存膨胀。

---

## Q4：虚拟线程还有哪些限制？

**A**：

| 限制 | 说明 |
|------|------|
| **不支持 stop/suspend** | 协作式调度，抛出 `UnsupportedOperationException` |
| **不支持设置优先级** | `setPriority()` 无效 |
| **始终是守护线程** | `setDaemon(false)` 无效 |
| **无法自定义载体线程池** | 固定使用 ForkJoinPool，大小等于 CPU 核数 |
| **不支持线程组操作** | `getThreadGroup()` 返回虚拟线程组 |
| **无法自定义栈大小** | 虚拟线程栈按需增长，不可手动设置 |

---

## Q5：interrupt() 在虚拟线程中的行为有变化吗？

**A**：

**有变化。** 这是虚拟线程容易踩坑的地方。

**平台线程中**：`interrupt()` 会立即唤醒 `sleep()`/`wait()`/`park()` 中的线程，并抛出 `InterruptedException`。

**虚拟线程中**：`interrupt()` **只设置中断标志位**，不直接唤醒被 `park()` 的虚拟线程：

```java
// 平台线程：interrupt() 立即唤醒
Thread t = new Thread(() -> {
    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        System.out.println("唤醒！");  // 立即执行
    }
});
t.start();
t.interrupt();  // 立即抛出 InterruptedException

// 虚拟线程：interrupt() 设置标志位，但不一定立即唤醒
Thread vt = Thread.startVirtualThread(() -> {
    LockSupport.park();
    // interrupt() 后可能还在 parking 状态
});
vt.interrupt();  // 设置中断标志，虚拟线程可能还在停车
```

**最佳实践**：不要用 `interrupt()` 取消虚拟线程任务，改用 **StructuredTaskScope 的作用域取消机制**（`ShutdownOnFailure`/`ShutdownOnSuccess`）。

---

## Q6：Thread.sleep() 和 LockSupport.park() 在虚拟线程中有什么区别？

**A**：

在虚拟线程中，两者都能触发挂起，但行为细节有区别：

| 对比 | `Thread.sleep()` | `LockSupport.park()` |
|------|-------------------|----------------------|
| 中断响应 | 抛出 `InterruptedException` | 只设置中断标志位，不抛异常 |
| 精确度 | JVM 调度影响（微秒级误差） | 需要手动循环检查，更高精度 |
| 使用场景 | 延迟等待（推荐） | 锁机制底层（如 `ReentrantLock`） |
| 可读性 | 高（`sleep(1000)` 一目了然） | 低（需要理解许可证机制） |

**推荐使用 `Thread.sleep()` 的原因**：

```java
// ✅ 推荐：清晰、可中断、精确
Thread.startVirtualThread(() -> {
    Thread.sleep(1000);  // 精确睡眠 1 秒，可中断
    process();
});

// ⚠️ 不推荐：不直观，中断处理麻烦
Thread.startVirtualThread(() -> {
    LockSupport.park();  // 为什么停车？需要看配套 unpark()
    // 中断只设置标志位，不抛异常
});
```

**注意**：`ReentrantLock` 内部使用 `LockSupport.park()`，开发者无需直接调用。



