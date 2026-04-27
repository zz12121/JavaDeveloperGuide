
# 虚拟线程挂起

## Q1：虚拟线程遇到阻塞操作会怎样？

**A**：

虚拟线程遇到阻塞操作（如网络 IO、`Thread.sleep()`、`BlockingQueue.take()` 等）时会**自动挂起**：

1. JVM 拦截阻塞操作
2. 将虚拟线程的执行状态保存到 Continuation 中
3. 虚拟线程与载体线程解绑（unmount）
4. 载体线程被释放，可以执行其他虚拟线程
5. IO 操作完成后，虚拟线程恢复（mount），挂载到可用载体线程继续执行

整个过程对开发者完全透明，无需使用回调或异步 API。

---

## Q2：哪些操作会触发虚拟线程挂起？

**A**：

- ✅ **网络 IO**：`Socket.read/write`、`HttpClient.send`
- ✅ **睡眠/等待**：`Thread.sleep()`、`LockSupport.park()`
- ✅ **阻塞队列**：`BlockingQueue.put/take`
- ✅ **Lock**：`ReentrantLock.lock()`（非 synchronized）
- ✅ **大部分文件 IO**：`FileChannel` 操作
- ⚠️ **synchronized**：JDK 21+ 大部分场景已优化，但极端情况可能 pinning
- ⚠️ **Native 方法**：JNI 中的阻塞操作可能导致 pinning

---

## Q3：怎么检测 pinning 问题？

**A**：

启动时加 JVM 参数：
```bash
-Djdk.tracePinnedThreads=short   # 简短输出
-Djdk.tracePinnedThreads=full    # 包含完整堆栈
```

发现 pinning 后，将 `synchronized` 替换为 `ReentrantLock` 即可解决。JDK 21+ 已自动优化大部分 synchronized 的 pinning 问题。

---

## Q4：LockSupport.park()/unpark() 在虚拟线程中的语义有变化吗？

**A**：

**有变化。** 这是虚拟线程中容易被忽略的重要细节。

**平台线程中**：`park()` 直接阻塞当前 OS 线程，涉及内核态切换（约 1-10μs）。

**虚拟线程中**：`park()` 被 JVM 拦截，虚拟线程挂起（unmount），**不阻塞载体线程**：

```java
// 平台线程：park() 阻塞 OS 线程
Thread t = new Thread(() -> {
    LockSupport.park();  // OS 线程被阻塞
});
LockSupport.unpark(t);  // 唤醒 OS 线程

// 虚拟线程：park() 挂起虚拟线程
Thread vt = Thread.startVirtualThread(() -> {
    LockSupport.park();  // JVM 拦截 → VT 挂起 → 载体线程释放
    System.out.println("resumed");
});
LockSupport.unpark(vt);  // 标记 VT 可运行 → 重新挂载
```

**行为对比**：

| 对比点 | 平台线程 | 虚拟线程 |
|--------|---------|---------|
| park() 后线程状态 | `WAITING`（OS 线程阻塞） | `PARKED`（JVM 管理） |
| 阻塞层级 | 内核态 | 用户态（不阻塞 OS 线程） |
| unpark() 提前调用 | 下次 park() 立即返回 | 行为相同（许可证机制不变） |
| 中断响应 | 设置中断标志位 | JDK 21+ 已统一语义 |

---

## Q5：虚拟线程挂起时，栈帧保存在哪里？

**A**：

虚拟线程挂起时，栈帧保存在 **JVM 堆内存中的 Continuation 对象**里，而不是 OS 线程栈。

```
虚拟线程挂起过程：

  执行中：VT 栈帧在载体线程的栈上（mounted）
       │
       ↓ 遇到阻塞操作
       │
  挂起：栈帧从载体线程栈 → 拷贝到 Continuation 对象（堆上）
       │
       ↓ unmount
       │
  挂起完成：VT 与载体线程解绑，Continuation 保留完整执行状态
       │
       ↓ IO 完成，重新 mount
       │
  恢复：Continuation 栈帧 → 拷贝回载体线程栈
       │
       ↓ 继续执行
       │
  完成
```

**为什么在堆上？**

- 载体线程的栈是 OS 管理的，JVM 无法随意操作
- 堆内存由 JVM 管理，可以 GC、可以跨线程传递
- 挂起后的虚拟线程就是一个普通的 Java 对象，可以被 GC 回收

**Continuation 的内存开销**：

- 初始约 512 字节（只有少量栈帧）
- 按需增长，最大不超过 `jdk.virtualThreadStackSize`（默认 512KB）
- 挂起时不占用载体线程栈空间




