
# 虚拟线程与ForkJoinPool

## Q1：虚拟线程的调度器和 ForkJoinPool 是什么关系？

**A**：**JDK 21 虚拟线程的调度器底层就是 ForkJoinPool。** 这是 JDK 21 的实现细节：

```
虚拟线程（VirtualThread）× N
    ↓ 挂载（mount）
平台线程（Carrier Thread）= ForkJoinWorkerThread × CPU核心数
    ↓ 工作窃取
ForkJoinPool（默认）
```

每个 CPU 核心对应一个 ForkJoinWorkerThread 作为 Carrier Thread。虚拟线程阻塞时卸载（unmount），Carrier Thread 从 ForkJoinPool 的 WorkQueue 中取下一个虚拟线程来执行。

这意味着 **ForkJoinPool 既是传统 ForkJoinTask 的执行器，也是虚拟线程的调度底座**。

---

## Q2：虚拟线程的 unmount 和 ForkJoinTask 的 join 有什么区别？

**A**：两者本质相同（都不让线程空转），但粒度和触发时机不同：

| 维度 | ForkJoinTask.join() | 虚拟线程 unmount |
|------|--------------------|-----------------|
| 触发条件 | 等待子任务完成 | 任何阻塞操作（I/O、Lock、sleep）|
| 阻塞粒度 | 整个任务级别 | 每个阻塞调用级别（更细） |
| 帮忙执行 | 执行 WorkQueue 中的其他 ForkJoinTask | 执行其他虚拟线程 |
| 恢复条件 | 子任务完成（可能被其他线程窃取）| I/O 完成（主动重新调度）|

```java
// ForkJoin：join 阻塞在任务级别
left.fork();
right.compute();
left.join(); // 等待左侧，可能帮忙执行其他 ForkJoinTask

// 虚拟线程：unmount 阻塞在调用级别
// 每个可能阻塞的操作（httpClient.get、db.query）都可能触发 unmount
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        String html = httpClient.get(url);     // unmount
        String data = db.query(sql);           // 再 unmount
        process(html, data);
    });
}
```

---

## Q3：JDK 21 为什么禁止在虚拟线程中使用 synchronized？

**A**：JDK 21 禁止在虚拟线程中使用 `synchronized` 关键字（会抛出 `IllegalMonitorStateException` 或导致性能问题），原因是 **synchronized 会阻塞整个 Carrier Thread**。

```
虚拟线程 V1（Carrier Thread 上）:
    synchronized(lock) {  // 持有 monitor
        doSomething();
    }
    ↓
    V1 阻塞（但未 unmount，因为持有 monitor）
    ↓
    其他虚拟线程 V2 无法使用同一 Carrier Thread
    ↓
    V2 被永久挂起，直到 V1 释放 monitor
```

`synchronized` 不支持 interruptibly，无法安全中断。解决方案是改用 `ReentrantLock`：

```java
// ✅ JDK 21+ 推荐
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 业务逻辑
} finally {
    lock.unlock();
}

// ✅ 或者用 synchronized 但保持临界区极小
synchronized (this) {
    // 极短操作
}
```

---

## Q4：虚拟线程和 ForkJoinPool 分别适合什么场景？

**A**：

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| CPU 密集型分治（归并排序、矩阵运算）| ForkJoinPool + ForkJoinTask | 手动控制拆分粒度，零开销 |
| 高并发 HTTP 请求、数据库查询 | 虚拟线程 | 阻塞自动 unmount，无需管理线程数 |
| 需要精确控制任务粒度 | ForkJoinPool | THRESHOLD 由开发者设定 |
| 简单的高并发 I/O（万级并发）| 虚拟线程 | 极低内存开销（1MB 栈 vs 1MB 堆）|
| Stream parallel + CPU 计算 | ForkJoinPool.commonPool | 内置并行流支持 |
| 混合场景（计算 + I/O）| 虚拟线程 + 独立计算线程池 | I/O 用虚拟线程，计算用 FJP |

**不要用虚拟线程的场景**：
- CPU 密集型计算（虚拟线程不减少延迟，只减少内存）
- 需要精确控制线程数的场景（线程池更可控）
- 对延迟极敏感的场景（FJP 的工作窃取更均衡）

---

## Q5：ForkJoinPool.ManagedBlocker 在 JDK 21+ 还有用吗？

**A**：**有用，但使用方式变了。**

ManagedBlocker 原本是 **JDK 7-17** 中在 ForkJoinPool 内安全执行阻塞操作的唯一正规方案。JDK 21+ 中：

1. **对于 ForkJoinTask**：ManagedBlocker 仍然有效，用于在 FJP 内执行阻塞操作而不降低并行度
2. **对于虚拟线程**：不需要 ManagedBlocker，因为虚拟线程阻塞时自动 unmount，Carrier Thread 自动切换到其他虚拟线程

```java
// 仍然适合：在 ForkJoinTask 中执行阻塞操作（JDK 17 及之前）
ForkJoinPool.ManagedBlocker blocker = new ForkJoinPool.ManagedBlocker() {
    @Override
    public boolean block() throws InterruptedException {
        lock.lock();
        try {
            // 阻塞操作
            return true;
        } finally {
            lock.unlock();
        }
    }
    @Override
    public boolean isReleasable() {
        return lock.tryLock();
    }
};
ForkJoinPool.managedBlock(blocker);

// JDK 21+ 更简单：用虚拟线程
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        lock.lock();  // 阻塞时自动 unmount
        try {
            // 业务逻辑
        } finally {
            lock.unlock();
        }
    });
}
```

---

```java
// 实战：虚拟线程实现高性能爬虫（对比 ForkJoin 方案）

// 方案1：ForkJoinPool（适合 CPU 计算）
class WebFetcher extends RecursiveTask<List<String>> {
    @Override
    protected List<String> compute() {
        if (urls.size() <= 10) {
            return fetchSequentially(urls); // 小批量直接请求
        }
        int mid = urls.size() / 2;
        WebFetcher left = new WebFetcher(urls.subList(0, mid));
        WebFetcher right = new WebFetcher(urls.subList(mid, urls.size()));
        left.fork();
        return Stream.concat(right.compute().stream(), left.join().stream())
                     .collect(Collectors.toList());
    }
}

// 方案2：虚拟线程（适合高并发 I/O）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> executor.submit(() -> fetch(url)))
        .collect(Collectors.toList());
    List<String> results = futures.stream()
        .map(f -> {
            try { return f.get(); }
            catch (Exception e) { return ""; }
        })
        .collect(Collectors.toList());
}

// 选择建议：
// - 爬虫瓶颈在网络 I/O → 虚拟线程（万级并发）
// - 爬虫后需要 CPU 计算解析 → ForkJoinPool（分治计算）
```
