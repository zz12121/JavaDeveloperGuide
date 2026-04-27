
# ForkJoinTask

## 核心结论

ForkJoinTask 是 ForkJoinPool 中的任务抽象基类，有两个常用子类：
- **RecursiveTask\<V\>**：有返回值的递归任务（实现 `compute()` 返回结果）
- **RecursiveAction**：无返回值的递归任务（实现 `compute()` 无返回值）

## 深度解析

### 类层次结构

```
ForkJoinTask<V>                    ← 抽象基类
├── RecursiveTask<V>               ← 有返回值
├── RecursiveAction                ← 无返回值
├── CountedCompleter<V>            ← 完成计数触发
└── ...（内部子类）
```

### 核心方法

| 方法 | 作用 |
|------|------|
| `fork()` | 将子任务推入当前线程的工作队列异步执行 |
| `join()` | 等待任务完成并获取结果（可能阻塞） |
| `compute()` | 子类实现，定义任务逻辑 |
| `invoke()` | 等价于 fork + join，同步执行 |
| `isDone()` | 任务是否完成 |
| `get()` | 继承自 Future，等待结果（支持超时和中断） |

### RecursiveTask vs RecursiveAction

| 特性 | RecursiveTask\<V\> | RecursiveAction |
|------|-------------------|-----------------|
| 返回值 | 有（V） | 无（void） |
| 典型场景 | 数组求和、归并排序 | 数组遍历、批量写入 |
| compute() | `return result` | 无返回值 |

### 任务状态

ForkJoinTask 内部维护状态字段，通过 `status` 字段（volatile int）表示：
- NORMAL（0）：正常完成
- CANCELLED（1）：被取消
- EXCEPTIONAL（-1）：执行异常
- SIGNAL（-2）：有线程在等待结果

## 代码示例

### RecursiveTask 示例（数组求和）

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int lo, hi;
    private static final int THRESHOLD = 10_000;

    SumTask(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    @Override
    protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += arr[i];
            return sum;
        }
        int mid = lo + (hi - lo) / 2;
        SumTask left = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();
        return right.compute() + left.join();
    }
}
```

### RecursiveAction 示例（数组元素翻倍）

```java
class DoubleTask extends RecursiveAction {
    private final long[] arr;
    private final int lo, hi;
    private static final int THRESHOLD = 10_000;

    DoubleTask(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    @Override
    protected void compute() {
        if (hi - lo <= THRESHOLD) {
            for (int i = lo; i < hi; i++) arr[i] *= 2;
        } else {
            int mid = lo + (hi - lo) / 2;
            invokeAll(new DoubleTask(arr, lo, mid),
                      new DoubleTask(arr, mid, hi));
        }
    }
}
```

# ForkJoinTask fork/join

## 核心结论

`fork()` 将子任务异步提交到当前线程的工作队列，`join()` 阻塞等待子任务完成并获取结果。最佳实践：一个子任务 fork，另一个子任务直接在当前线程 compute，避免不必要的入队开销。

## 深度解析

### fork() 源码逻辑

```java
public final ForkJoinTask<V> fork() {
    Thread t = Thread.currentThread();
    if (t instanceof ForkJoinWorkerThread) {
        // 工作线程：推入自己的工作队列
        ((ForkJoinWorkerThread) t).workQueue.push(this);
    } else {
        // 非工作线程（外部提交）：推入 commonPool
        ForkJoinPool.commonPool().externalPush(this);
    }
    return this;
}
```

### join() 源码逻辑

```java
public final V join() {
    int s;
    // 如果任务未完成（状态 <= 0）
    if ((s = status) >= 0)
        s = doJoin();  // 执行或等待任务完成
    // 根据状态返回结果或抛异常
    return reportResult(s);
}
```

**doJoin()** 的关键逻辑：

1. 先尝试**自己执行**（如果尚未开始）
2. 如果已完成，直接返回
3. 如果未完成，当前线程**执行其他任务**（help other tasks），同时等待目标任务完成
4. 期间可被其他线程窃取执行

### 执行模式对比

```
方式一：left.fork() + right.compute()  ← 推荐 ✅
┌─────────────────────┐
│ 当前线程            │
│  ├─ left.fork()    │ → 推入队列，可能被其他线程窃取
│  └─ right.compute() │ → 当前线程直接执行
│  left.join()        │ → 等待 left 完成
└─────────────────────┘

方式二：left.fork() + right.fork()  ← 不推荐 ❌
┌─────────────────────┐
│ 当前线程            │
│  ├─ left.fork()    │ → 入队
│  ├─ right.fork()   │ → 入队
│  │ 队列空了！       │
│  └─ 当前线程阻塞等待 │ → 浪费，需等窃取或从队列取回
└─────────────────────┘
```

### invokeAll() 工具方法

```java
// 等价于：task1.fork(); task2.fork(); task2.join(); task1.join()
// 或者：invokeAll(task1, task2)
static void invokeAll(ForkJoinTask<?> t1, ForkJoinTask<?> t2)
```

内部实现会合理分配执行，比手动 fork 两个任务更高效。
### THRESHOLD 如何合理设置

THRESHOLD（任务拆分阈值）是 ForkJoin 性能的关键参数，设置不当会适得其反：

**经验公式**：

```java
// CPU 密集型：THRESHOLD ≈ 数据总量 / (并行度 × 4)
int threshold = Math.max(1,
    array.length / (ForkJoinPool.getCommonPoolParallelism() * 4)
);

// 动态阈值（根据并行度自动调整）
int threshold = Math.max(1, 10_000); // 保底值
```

**4 的含义**：超分因子（over-partition factor）。设置 4 倍数量的子任务，给工作窃取留余量。如果拆分不够细，负载均衡会变差；如果拆分太细，fork/join 开销超过计算收益。

**判断标准：单次 compute 耗时应在 100μs ~ 10ms 之间**：

| 阈值太大 | 阈值太小 |
|---------|---------|
| 任务粒度粗，负载均衡差 | fork/join 开销 > 计算收益 |
| 并行度浪费 | 队列竞争加剧 |
| 极端：等于数据总量 → 退化为串行 | 极端：每个元素一个任务 → 大量 fork 开销 |

**实测方法**：

```java
public class ThresholdTuner {
    public static void main(String[] args) {
        long[] array = new long[10_000_000];
        Arrays.fill(array, 1);

        int[] thresholds = {1_000, 5_000, 10_000, 50_000, 100_000, 500_000};

        for (int threshold : thresholds) {
            long start = System.nanoTime();
            ForkJoinPool pool = new ForkJoinPool();
            long sum = pool.invoke(new TunedSumTask(array, 0, array.length, threshold));
            long end = System.nanoTime();
            System.out.printf("THRESHOLD=%7d 耗时=%.3fs%n", threshold, (end-start)/1e9);
        }
    }
}
```

**典型场景参考值**：

| 场景 | 建议 THRESHOLD |
|------|----------------|
| 数组求和（long[]）| 10,000 ~ 100,000 |
| 归并排序 | 1,000 ~ 10,000（比较操作更复杂）|
| 目录遍历 | 子目录数 10 ~ 50 |
| 文件哈希计算 | 1MB ~ 10MB（每个任务处理的粒度）|

### 实战场景：大 List 分批写入数据库

```java
/**
 * 将大 List 并行分批写入数据库（RecursiveAction，无返回值）
 * 适用于数据迁移、批量同步等场景
 */
public class BatchWriteTask extends RecursiveAction {
    private final List<Entity> data;
    private final int batchSize;
    private final EntityRepository repository;
    private static final int BATCH_THRESHOLD = 5000;

    public BatchWriteTask(List<Entity> data, int batchSize, EntityRepository repository) {
        this.data = data;
        this.batchSize = batchSize;
        this.repository = repository;
    }

    @Override
    protected void compute() {
        if (data.size() <= BATCH_THRESHOLD) {
            // 数据量小，直接写入
            repository.saveAll(data);
            return;
        }

        // 数据量大，拆分为多个批次并行写入
        int mid = data.size() / 2;
        List<Entity> left = data.subList(0, mid);
        List<Entity> right = data.subList(mid, data.size());

        // 使用 invokeAll，同时执行两个子任务
        invokeAll(
            new BatchWriteTask(left, batchSize, repository),
            new BatchWriteTask(right, batchSize, repository)
        );
        // invokeAll 内部自动 fork + join，无需手动管理
    }
}

// 使用
ForkJoinPool pool = new ForkJoinPool();
pool.invoke(new BatchWriteTask(hugeList, 1000, repository));
```

## 关联知识点