# CountedCompleter

## 核心结论

CountedCompleter 是 ForkJoinTask 的子类，比 RecursiveTask/RecursiveAction 更强大：它通过 **pending count（待完成计数）** 机制自动触发回调，不需要显式 join 等待子任务。适合"**扇出-合并**"模式（如目录遍历、爬虫、任务图执行）。

## 深度解析

### 类层次结构

```
ForkJoinTask<V>
├── RecursiveTask<V>        ← 需要 join 收集结果
├── RecursiveAction         ← 无返回值
└── CountedCompleter<V>     ← 完成计数触发回调（更高级）
```

### 核心概念：pending count

每个 CountedCompleter 维护一个待完成子任务计数：

```
当前任务 pending = N
    ↓ fork() 子任务时
当前任务 pending = N + M（M个子任务）
    ↓ 每个子任务完成（调用 tryComplete 或 onCompletion）
当前任务 pending--
    ↓ pending == 0 时
自动触发 onCompletion() 回调
```

### 核心方法

| 方法 | 作用 |
|------|------|
| `addToPendingCount(int delta)` | 增加待完成计数 |
| `compareAndSetPendingCount(int expect, int update)` | CAS 设置计数 |
| `decrementPendingCountUnlessZero()` | 原子递减（线程安全） |
| `tryComplete()` | 递减 pending count，为 0 时触发 onCompletion |
| `onCompletion(CountedCompleter<?> caller)` | 所有子任务完成后回调（需子类实现） |
| `onExceptionalCompletion(Throwable ex, CountedCompleter<?> caller)` | 异常完成时的回调（可选覆盖） |
| `getPendingCount()` | 获取当前 pending count |

### 三种触发回调的方式

```java
// 方式1：tryComplete（常用）
protected void compute() {
    for (Item item : items) {
        if (isSmall(item)) {
            process(item); // 小任务直接处理，不拆
        } else {
            addToPendingCount(1);
            new SubTask(this, item).fork(); // 拆分子任务
        }
    }
    tryComplete(); // 尝试完成当前任务
}

// 方式2：propagateCompletion（无条件触发）
// 立即将完成状态向上传递给父任务，跳过 tryComplete 的条件判断

// 方式3：quickerQuietComplete（快速完成）
// 直接将 pending count 设为 0，无条件触发回调
```

### 对比 RecursiveTask vs CountedCompleter

| 特性 | RecursiveTask | CountedCompleter |
|------|--------------|-----------------|
| 结果收集 | 手动 join | 自动 onCompletion 回调 |
| 子任务依赖 | 需要 join 等待 | 不需要 join，计数触发 |
| 适用模式 | 递归求和、排序 | 扇出遍历、批量处理 |
| 异常传播 | join 时抛 RuntimeException | 同样机制 |
| 适用场景 | 结果需要合并 | 只需执行动作或触发后续处理 |

### 实战场景：目录树文件统计

```java
/**
 * 并行遍历目录树，统计文件数量和总大小
 * 不需要手动 join，通过 pending count 自动触发合并
 */
public class FileCounter extends CountedCompleter<Void> {
    private final File dir;
    private final Map<String, Long> result; // 共享结果（需外部同步）

    public FileCounter(CountedCompleter<?> parent, File dir, Map<String, Long> result) {
        super(parent);
        this.dir = dir;
        this.result = result;
    }

    public FileCounter(File dir, Map<String, Long> result) {
        this(null, dir, result);
    }

    @Override
    public void compute() {
        File[] children = dir.listFiles();
        if (children == null || children.length == 0) {
            tryComplete(); // 空目录，直接完成
            return;
        }

        // 遍历子文件和子目录
        for (File child : children) {
            if (child.isDirectory()) {
                // 目录：拆分为子任务
                addToPendingCount(1);          // 增加待完成计数
                new FileCounter(this, child, result).fork(); // 异步执行
            } else {
                // 文件：直接统计
                synchronized (result) {
                    result.merge("count", 1L, Long::sum);
                    result.merge("size", child.length(), Long::sum);
                }
            }
        }
        tryComplete(); // 当前层完成，等待子任务
    }

    @Override
    public void onCompletion(CountedCompleter<?> caller) {
        // 所有子任务完成后自动调用
        // 可在此处执行额外的合并操作
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        Map<String, Long> result = new ConcurrentHashMap<>();
        result.put("count", 0L);
        result.put("size", 0L);

        long start = System.nanoTime();
        pool.invoke(new FileCounter(new File("C:\\"), result));
        long end = System.nanoTime();

        System.out.println("文件数: " + result.get("count"));
        System.out.println("总大小: " + result.get("size") + " bytes");
        System.out.println("耗时: " + (end - start) / 1_000_000 + " ms");
    }
}
```

### 实战场景：批量数据库写入

```java
/**
 * 并行将大 List 分批写入数据库
 * 用 CountedCompleter 管理批次，完成后触发回调
 */
public class BatchDBWriter<T> extends CountedCompleter<Void> {
    private final List<T> data;
    private final int batchSize;
    private final Consumer<List<T>> batchWriter;
    private static final int BATCH_THRESHOLD = 1000;

    public BatchDBWriter(CountedCompleter<?> parent, List<T> data,
                         int batchSize, Consumer<List<T>> batchWriter) {
        super(parent);
        this.data = data;
        this.batchSize = batchSize;
        this.batchWriter = batchWriter;
    }

    public BatchDBWriter(List<T> data, int batchSize,
                         Consumer<List<T>> batchWriter) {
        this(null, data, batchSize, batchWriter);
    }

    @Override
    public void compute() {
        if (data.size() <= BATCH_THRESHOLD) {
            // 数据量小，直接写入
            batchWriter.accept(data);
            tryComplete();
            return;
        }

        // 数据量大，拆分写入
        int mid = data.size() / 2;
        List<T> left = data.subList(0, mid);
        List<T> right = data.subList(mid, data.size());

        addToPendingCount(1);
        invokeAll(
            new BatchDBWriter<>(this, left, batchSize, batchWriter),
            new BatchDBWriter<>(this, right, batchSize, batchWriter)
        );
        // 无需手动 join，tryComplete 在 invokeAll 内部自动处理
    }

    @Override
    public void onCompletion(CountedCompleter<?> caller) {
        // 所有批次写入完成后可执行收尾操作（如刷新连接、记录日志）
    }
}
```

### 注意事项

1. **必须调用 tryComplete()**：否则 onCompletion 永远不会被触发
2. **addToPendingCount + fork() 配对**：每 fork 一个子任务就要 addToPendingCount
3. **线程安全**：CountedCompleter 本身不保证线程安全，结果收集需要同步（如 ConcurrentHashMap）
4. **异常取消**：子任务异常时，其他子任务同样会被取消

## 关联知识点

- RecursiveTask：需要 join 收集结果，适合递归求和
- CountedCompleter：不需要 join，通过计数触发，适合扇出遍历
- invokeAll：在 CountedCompleter 中同样适用，自动管理 pending count
