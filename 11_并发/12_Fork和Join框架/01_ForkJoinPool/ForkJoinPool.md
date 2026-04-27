
# ForkJoinPool

## 核心结论

ForkJoinPool 是 JDK7 引入的**分治并行执行框架**，专为**可拆分的递归任务**设计。核心思想：将大任务拆分（Fork）为小任务并行执行，最后合并（Join）结果。采用**工作窃取算法**提升线程利用率，每个线程维护双端队列，空闲时从其他线程尾部"偷"任务。

## 深度解析

### 设计思想

```
大任务
├── 子任务A ──→ 拆分 ──→ A1, A2, A3
├── 子任务B ──→ 拆分 ──→ B1, B2
└── 子任务C ──→ 拆分 ──→ C1, C2, C3
                    ↓
              合并所有子结果 → 最终结果
```

**分治（Divide and Conquer）**：
1. **Fork**：将任务拆分为更小的子任务
2. **执行**：子任务并行执行
3. **Join**：等待子任务完成，合并结果

### 核心组件

| 组件 | 作用 |
|------|------|
| **ForkJoinPool** | 线程池，管理工作线程 |
| **ForkJoinWorkerThread** | 工作线程，每个有独立的双端队列 |
| **ForkJoinTask** | 任务基类，定义 fork/join 操作 |
| **WorkQueue** | 双端任务队列，支持 LIFO/FIFO |

### 与普通线程池的本质区别

| 特性 | ThreadPoolExecutor | ForkJoinPool |
|------|-------------------|--------------|
| 任务来源 | 外部提交 | 自身产生子任务 |
| 工作模式 | 共享工作队列 | 每线程独立队列 |
| 任务关系 | 独立任务 | 父子任务，有依赖 |
| 空闲处理 | 线程阻塞等待 | 工作窃取 |
| 适用场景 | 通用并发 | 分治递归 |

### 关键参数

```java
// 默认并行度 = CPU 核心数 - 1
ForkJoinPool pool = new ForkJoinPool();

// 指定并行度
ForkJoinPool pool = new ForkJoinPool(4);

// 获取公共池（JDK8+）
ForkJoinPool common = ForkJoinPool.commonPool();
```

### WorkQueue 数组与工作窃取深度解析

ForkJoinPool 内部维护一个 **WorkQueue[] 数组**，这是理解工作窃取的关键：

```
ForkJoinPool 内部结构：
┌─────────────────────────────────────────────────────────┐
│  WorkQueue[] queues                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │ q[0]    │ │ q[1]    │ │ q[2]    │ │ q[3]    │ ...  │
│  │ 外部队列 │ │ W1队列  │ │ 外部队列 │ │ W2队列  │      │
│  │ 外部提交 │ │ LIFO    │ │ 外部提交 │ │ LIFO    │      │
│  │ 偶数索引 │ │ 奇数索引 │ │         │ │         │      │
│  └────┬────┘ └────┬────┘ └─────────┘ └────┬────┘      │
│       │            │                       │           │
│       ↓            ↓                       ↓           │
│    push/pop     push/pop(头)           steal(尾)       │
│    外部线程     自身线程操作             窃取者操作用尾   │
└─────────────────────────────────────────────────────────┘
```

**索引分配规则**：
- **偶数索引（0, 2, 4...）**：外部提交队列，外部线程（非 ForkJoinWorkerThread）提交的任务放在这里
- **奇数索引（1, 3, 5...）**：工作线程队列，每个 ForkJoinWorkerThread 拥有一个独立队列

**LIFO vs FIFO 的设计意图**：

| 操作 | 方向 | 模式 | 谁操作 | 目的 |
|------|------|------|--------|------|
| push | 头部入队 | LIFO | 自身线程（fork 子任务时）| 数据局部性：刚创建的任务共享父任务数据，热缓存 |
| pop | 头部出队 | LIFO | 自身线程（执行任务时）| 缓存友好：处理自己刚放入的任务 |
| steal | 尾部出队 | FIFO | 窃取线程 | 减少竞争：窃取"最老"的大粒度任务，一次干活更多 |

**头尾分离的核心价值**：同一个队列的 push/pop 和 steal 操作竞争不同位置（头 vs 尾），避免了同一位置的 CAS 竞争，这是 ForkJoinPool 高性能的秘密。

### ManagedBlocker：在 ForkJoinPool 内安全执行阻塞操作

ForkJoinPool 的工作线程数量有限（默认 CPU 核数 - 1），如果任务中包含阻塞操作（Lock、Semaphore、CountDownLatch 等），线程被阻塞后无法处理窃取，整体并行度下降。**ManagedBlocker 是 JDK 官方提供的解决方案**。

```java
ForkJoinPool.ManagedBlocker blocker = new ForkJoinPool.ManagedBlocker() {
    // 阻塞操作是否已完成
    @Override
    public boolean isReleasable() {
        return lock.tryLock();  // 能获取锁则不需要阻塞
    }

    // 执行阻塞操作，返回 true 表示正常完成
    @Override
    public boolean block() throws InterruptedException {
        lock.lock();
        try {
            // 临界区操作
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
```

**原理**：ForkJoinPool 检测到工作线程即将阻塞时，会**临时补充一个补偿线程**，保持总并行度不变：

```
工作线程 W1 执行 ManagedBlocker.block()
    ↓ 检测到即将阻塞
ForkJoinPool 临时创建补偿线程 W_1'
    ↓
W1 继续执行阻塞操作（W1 和 W_1' 同时运行）
    ↓
阻塞操作完成
    ↓
补偿线程 W_1' 自动退出或回归空闲
```

**适用场景**：
- ForkJoinTask 中必须持有锁的临界区
- 信号量限流（Semaphore.acquire()）
- CountDownLatch.await()
- 任何在 ForkJoinPool 中无法避免的阻塞操作

**注意**：JDK 21 虚拟线程场景下，应优先考虑用虚拟线程替代 ManagedBlocker，因为虚拟线程阻塞时自动 unmount。

## 代码示例

```java
import java.util.concurrent.*;

// 计算数组求和示例
public class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 10_000;

    public SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // 任务足够小，直接计算
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }
        // 拆分为两个子任务
        int mid = start + length / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        
        left.fork();  // 异步执行左半部分
        return right.compute() + left.join();  // 右半部分本线程执行 + 等待左半部分
    }

    public static void main(String[] args) {
        long[] array = new long[100_000];
        Arrays.fill(array, 1);
        
        ForkJoinPool pool = new ForkJoinPool();
        long result = pool.invoke(new SumTask(array, 0, array.length));
        System.out.println("Sum: " + result);  // 100000
    }
}
```

# ForkJoinPool.commonPool

## 核心结论

`ForkJoinPool.commonPool()` 是 JDK8 引入的**全局共享 ForkJoinPool**，并行流（parallel stream）、CompletableFuture 默认都使用它。默认并行度为 `CPU 核心数 - 1`，所有使用 commonPool 的组件共享线程资源，需要注意**线程污染**和**阻塞风险**。

## 深度解析

### 获取方式

```java
// 获取公共池（懒加载，首次使用时创建）
ForkJoinPool pool = ForkJoinPool.commonPool();

// 也可以通过系统属性调整并行度
// -Djava.util.concurrent.ForkJoinPool.common.parallelism=8
```

### 默认参数

|参数|值|说明|
|---|---|---|
|并行度|CPU 核心数 - 1|至少 1|
|工作线程|懒创建|需要时才启动|
|线程模式|普通线程|非守护线程，阻止 JVM 退出|

### 使用 commonPool 的组件

|组件|说明|
|---|---|
|`stream().parallel()`|并行流默认使用 commonPool|
|`CompletableFuture.supplyAsync()`|无 Executor 参数时默认 commonPool|
|`Arrays.parallelSort()`|并行排序|

### 线程污染问题

```java
// ❌ 危险：在 parallel stream 中执行阻塞操作
list.parallelStream().forEach(item -> {
    httpClient.get(item.getUrl());  // 阻塞 I/O！
    // 会占满 commonPool 所有线程，整个应用的并行操作都卡住
});

// ✅ 安全：为阻塞操作指定独立线程池
ExecutorService ioPool = Executors.newFixedThreadPool(10);
list.parallelStream()
    .forEach(item -> ioPool.submit(() -> {
        httpClient.get(item.getUrl());
    }));
```

### 自定义 ForkJoinPool 执行并行流

```java
// ✅ 用自定义池执行并行流，避免污染 commonPool
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() -> {
    int result = list.parallelStream()
        .mapToInt(i -> heavyCompute(i))
        .sum();
    System.out.println(result);
}).get();
```

### commonPool 最佳实践

1. **CPU 密集型短任务**：可以放心使用 commonPool
2. **I/O 阻塞操作**：必须使用独立的 ThreadPoolExecutor
3. **长时间计算**：考虑自定义 ForkJoinPool，避免阻塞其他组件
4. **系统属性调优**：生产环境可通过 `-D` 调整并行度
### 实战场景：目录树文件统计

```java
/**
 * 并行统计目录树文件数量和总大小
 * 典型分治场景：父任务遍历目录，子任务递归处理子目录
 */
public class FileSizeTask extends RecursiveTask<Long> {
    private final File dir;
    private static final int DIR_THRESHOLD = 10; // 子目录数超过此值才拆分

    public FileSizeTask(File dir) {
        this.dir = dir;
    }

    @Override
    protected Long compute() {
        File[] children = dir.listFiles();
        if (children == null || children.length == 0) {
            return 0L;
        }

        // 统计文件大小
        long fileSize = 0L;
        List<FileSizeTask> subTasks = new ArrayList<>();

        for (File child : children) {
            if (child.isFile()) {
                fileSize += child.length();
            } else if (child.isDirectory()) {
                subTasks.add(new FileSizeTask(child));
            }
        }

        // 子目录数超过阈值才并行，否则串行更高效
        if (subTasks.size() <= DIR_THRESHOLD) {
            long total = fileSize;
            for (FileSizeTask task : subTasks) {
                total += task.invoke(); // 串行合并
            }
            return total;
        }

        // 大量子目录：fork 所有子任务并行处理
        long total = fileSize;
        for (FileSizeTask task : subTasks) {
            task.fork();
        }
        for (FileSizeTask task : subTasks) {
            total += task.join(); // 合并结果
        }
        return total;
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        long size = pool.invoke(new FileSizeTask(new File("C:\\workspace")));
        System.out.println("Total size: " + size + " bytes");
    }
}
```

## 关联知识点
