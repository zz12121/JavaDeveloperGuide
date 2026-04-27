
# ForkJoinPool

## Q1：什么是 ForkJoinPool？它和普通线程池有什么区别？

**A**：ForkJoinPool 是 JDK7 引入的分治并行执行框架，专为可拆分递归任务设计。

**核心区别**：
- **ThreadPoolExecutor**：线程从共享队列取任务，任务间独立，适合通用并发
- **ForkJoinPool**：每个线程有独立双端队列，任务通过 fork 产生子任务，join 合并结果，通过工作窃取提升利用率

ForkJoinPool 适合**大任务可拆分为相似子任务**的场景（如数组求和、归并排序、文件目录遍历），不适合 I/O 密集或任务间有复杂依赖的场景。

---

## Q2：ForkJoinPool 的默认并行度是多少？

**A**：默认并行度为 `Runtime.getRuntime().availableProcessors() - 1`（CPU 核心数 - 1），可以通过构造函数指定：

```java
new ForkJoinPool(4)  // 并行度为 4
```

JDK8+ 提供了 `ForkJoinPool.commonPool()` 公共池，并行度同样为 CPU 核心数 - 1，被并行流等场景默认使用。

---

## Q3：ForkJoinPool 的工作窃取是如何减少线程空闲的？

**A**：每个工作线程维护自己的双端队列（WorkQueue），正常工作时从**队列头部**取任务（LIFO），执行过程中产生的子任务推入头部。

当某个线程队列空了（空闲），它会从**其他线程队列的尾部**"偷"任务（FIFO）来执行。这样：
- 活跃线程的子任务会被空闲线程"偷走"执行
- 减少了线程阻塞等待，提高了 CPU 利用率
- 双端队列的头尾分离避免了竞争冲突


# ManagedBlocker

## Q1：为什么 ForkJoinPool 不适合执行阻塞操作？ManagedBlocker 是怎么解决这个问题的？

**A**：ForkJoinPool 的工作线程数量有限（默认 CPU 核数 - 1），如果任务阻塞：

```
正常情况（4核 CPU，4个工作线程）：
W1[任务] W2[任务] W3[任务] W4[任务]  → 并行度 = 4

W1 阻塞（sleep 1秒）：
W1[阻塞] W2[任务] W3[任务] W4[任务]  → 并行度 = 3，W1 空转等
```

**ManagedBlocker 的解决方案**：ForkJoinPool 检测到即将阻塞时，**临时补充一个补偿线程**，保持总并行度不变：

```java
ForkJoinPool pool = new ForkJoinPool();
pool.execute(() -> {
    ForkJoinPool.ManagedBlocker blocker = new ForkJoinPool.ManagedBlocker() {
        @Override
        public boolean isReleasable() {
            return lock.tryLock();  // 能获取锁就不阻塞
        }
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
    };
    try {
        ForkJoinPool.managedBlock(blocker);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});
```

---

## Q2：ManagedBlocker 的 isReleasable() 和 block() 分别起什么作用？

**A**：ManagedBlocker 的核心逻辑：

```
managedBlock() 执行流程：
    ↓
isReleasable() 返回 true？
    ↓ yes
直接返回（不阻塞）
    ↓ no
block() 执行阻塞操作
    ↓
block() 返回 true？
    ↓ yes
返回（正常完成）
    ↓ no
循环重试 isReleasable() → block() → ...
```

**isReleasable()**：轻量检查，判断阻塞条件是否已满足（如 tryLock 成功则不需要阻塞）

**block()**：执行真正的阻塞操作，返回 true 表示正常完成

```java
// 典型实现：isReleasable 做非阻塞检查，block 做阻塞等待
ForkJoinPool.ManagedBlocker blocker = new ForkJoinPool.ManagedBlocker() {
    private final CountDownLatch latch = someLatch;

    @Override
    public boolean isReleasable() {
        return latch.getCount() == 0; // 计数为0则不需要等待
    }

    @Override
    public boolean block() throws InterruptedException {
        return latch.await(1, TimeUnit.HOURS); // 最多等1小时
    }
};
```

---

## Q3：JDK 21 虚拟线程出现后，还需要 ManagedBlocker 吗？

**A**：**分场景**：

- **ForkJoinTask 中执行阻塞操作**：ManagedBlocker 仍然需要，因为虚拟线程无法直接用于 ForkJoinPool 的工作线程
- **高并发 I/O 场景**：直接用虚拟线程（JDK 21+），不需要 ManagedBlocker，因为虚拟线程自动 unmount

```java
// 场景1：ForkJoinTask 中必须用 ManagedBlocker（JDK 17 及之前）
class TaskWithLock extends RecursiveTask<Void> {
    @Override
    protected Void compute() {
        ForkJoinPool.managedBlock(blocker); // 官方方案
        return null;
    }
}

// 场景2：JDK 21+ 高并发 I/O，直接用虚拟线程
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        lock.lock();  // 自动 unmount，不需要 ManagedBlocker
        try { /* ... */ }
        finally { lock.unlock(); }
    });
}
```

---

# ForkJoinPool.commonPool

## Q1：什么是 ForkJoinPool.commonPool？有哪些组件在使用它？

**A**：commonPool 是 JDK8 引入的全局共享 ForkJoinPool 实例，默认并行度为 CPU 核心数 - 1。以下组件默认使用它：

- `stream().parallel()` — 并行流
- `CompletableFuture.supplyAsync()` / `runAsync()` — 无 Executor 时
- `Arrays.parallelSort()` — 并行排序

所有这些操作**共享同一个线程池**，线程资源互相影响。

---

## Q2：使用 commonPool 有什么风险？

**A**：最大风险是**线程污染**：

1. 在 parallel stream 中执行阻塞 I/O（HTTP 请求、数据库查询），会占满所有工作线程
2. 其他使用 commonPool 的组件（如 CompletableFuture）会因无线程可用而被阻塞
3. 影响范围是**整个 JVM 进程**，不仅是当前业务

解决方案：

- 阻塞操作用独立 ThreadPoolExecutor
- 长时间计算用自定义 ForkJoinPool
- 生产环境通过 `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N` 调整并行度

---

## Q3：如何让并行流使用自定义的 ForkJoinPool？

**A**：通过 `ForkJoinPool.submit()` 提交并行流操作：

```java
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() -> {
    list.parallelStream().forEach(item -> process(item));
}).get();
```

原理：parallel stream 内部通过 `ForkJoinTask.getPool()` 获取当前线程所属的 ForkJoinPool。在自定义池的线程中执行时，自然使用该池。