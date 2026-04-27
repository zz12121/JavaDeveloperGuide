# 虚拟线程与ForkJoinPool

## 核心结论

**JDK 21 虚拟线程（Virtual Thread）的调度器底层就是 ForkJoinPool。** 虚拟线程挂载到平台线程（Carrier Thread）上运行，而这些平台线程本身就是 ForkJoinWorkerThread。理解这一点，就理解了虚拟线程与 ForkJoinPool 的本质关系。

## 深度解析

### 虚拟线程调度架构

```
虚拟线程（VirtualThread）  × 10000
    └─ 挂载到 → 平台线程（ForkJoinWorkerThread） × CPU核心数
                   └─ 运行在 → ForkJoinPool（调度器）
```

**三层调度架构**：

```
第一层：虚拟线程挂载器（MountAppender）
  虚拟线程 → 挂载到 Carrier Thread（平台线程） → 运行代码
  阻塞时 → 卸载（unmount），挂起虚拟线程，Carrier Thread 去执行其他虚拟线程

第二层：ForkJoinPool 工作窃取调度器
  平台线程就是 ForkJoinWorkerThread
  从 WorkQueue 头取任务（LIFO），从其他队列尾偷任务（FIFO）

第三层：OS 线程调度器
  ForkJoinWorkerThread → 映射到 OS 原生线程 → CPU 核心执行
```

### 虚拟线程与 ForkJoinPool 的关键对应

| 维度 | 传统 ForkJoin 任务 | 虚拟线程 |
|------|------------------|---------|
| 执行单元 | ForkJoinTask（需要手动拆分） | VirtualThread（自动调度） |
| 阻塞时行为 | join() 帮助执行其他任务 | unmount 卸载，Carrier 线程接新虚拟线程 |
| 任务粒度 | 开发者手动设 THRESHOLD | JVM 自动管理 |
| 适用场景 | CPU 密集型分治计算 | 高并发 I/O |
| 与 FJP 关系 | 直接提交 ForkJoinTask | 虚拟线程的调度器 = ForkJoinPool |

### ForkJoinPool 的双重身份

```
ForkJoinPool 角色一：任务执行器（传统场景）
  submit(ForkJoinTask) → WorkQueue → ForkJoinWorkerThread 执行

ForkJoinPool 角色二：虚拟线程调度器（JDK 21+）
  VirtualThread → MountAppender 挂载 → ForkJoinWorkerThread 执行
                 阻塞时卸载 → 其他虚拟线程使用同一线程
```

### unmount vs join：本质相同，粒度不同

**传统 ForkJoin 任务阻塞（join）**：
```java
Long result = left.join();  // 当前线程阻塞
// 底层：线程去执行 WorkQueue 中的其他任务（help），不空转
```

**虚拟线程阻塞（unmount）**：
```java
// 虚拟线程中执行 I/O 操作
String data = httpClient.get(url);  // 阻塞
// 底层：Carrier Thread 卸载该虚拟线程，去执行其他虚拟线程的任务
```

**对比图**：

```
ForkJoin 任务（join）：
┌────────────────────────────────────┐
│ WorkerThread                       │
│  left.join() 阻塞                  │
│    ↓ 帮忙执行                      │
│  执行 WorkQueue 中其他任务          │
└────────────────────────────────────┘

虚拟线程（unmount）：
┌────────────────────────────────────┐
│ CarrierThread (ForkJoinWorker)     │
│  VirtualThread1 阻塞               │
│    ↓ unmount 卸载                  │
│  VirtualThread2 使用同一线程       │
└────────────────────────────────────┘
```

两者本质都是**不让线程空转**，只是虚拟线程的粒度更细（到每个 I/O 操作），ForkJoin 的粒度在任务级别。

### JDK 21 禁止在虚拟线程中使用 synchronized

这是影响 ForkJoinPool 的关键限制：

```java
// ❌ JDK 21+ 虚拟线程中禁止使用 synchronized
void badMethod() {
    synchronized (lock) {  // 抛出 Jeffrey Dean 锁
        // do something
    }
}

// ✅ 改用 ReentrantLock
void goodMethod() {
    lock.lock();
    try {
        // do something
    } finally {
        lock.unlock();
    }
}
```

**原因**：synchronized 在虚拟线程中会阻塞整个 Carrier Thread（平台线程），导致所有共享该 Carrier 的虚拟线程都被阻塞。ReentrantLock 支持 **interruptibly**，可以安全中断。

**对 ForkJoinPool.ManagedBlocker 的影响**：
JDK 21+ 中，`ForkJoinPool.ManagedBlocker` 内部已改用 ReentrantLock，不再使用 synchronized。

### 混合使用策略

```
场景                          推荐方案
─────────────────────────────────────────────
CPU 密集型分治计算              ForkJoinPool + ForkJoinTask
高并发 I/O（HTTP/DB）           虚拟线程 + 独立线程池
需要手动控制任务拆分粒度         ForkJoinPool + RecursiveTask
简单的高并发 I/O 场景            虚拟线程（无需手动管理）
```

### 迁移指南：ForkJoin → 虚拟线程

```java
// 旧方案：ForkJoinTask 手动分治
ForkJoinPool pool = new ForkJoinPool();
pool.invoke(new DirSizeTask(rootDir));

// 新方案：虚拟线程 + 递归遍历
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<Long> future = pool.submit(() -> countDir(rootDir));
    // 虚拟线程自动管理并发，阻塞时自动让出 Carrier Thread
}

// 两者的关键区别：
// - ForkJoinPool：任务必须手动拆分（THRESHOLD），控制粒度在开发者
// - 虚拟线程：JVM 自动调度，适合 I/O 密集，粒度由 JVM 控制
```

## 关联知识点

- ForkJoinPool：虚拟线程的底层调度器
- ForkJoinWorkerThread：虚拟线程挂载的 Carrier Thread
- ManagedBlocker：JDK 17 中在 FJP 内执行阻塞操作的正规方案
- 虚拟线程 unmount：与 join() 的 help 机制本质相同
