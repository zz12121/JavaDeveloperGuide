
# ForkJoinTask

## Q1：ForkJoinTask 有哪些常用子类？分别适用什么场景？

**A**：
- **RecursiveTask\<V\>**：有返回值的递归任务，`compute()` 返回结果。适用于**需要汇总结果**的场景，如数组求和、归并排序、统计文件行数
- **RecursiveAction**：无返回值的递归任务，`compute()` 返回 void。适用于**只需执行动作**的场景，如数组遍历处理、批量文件操作

另外还有一个 **CountedCompleter**，支持完成计数后触发回调，适用于更复杂的任务编排。

---

## Q2：fork() 和 join() 分别做什么？为什么推荐 left.fork() + right.compute() 而不是两个都 fork？

**A**：
- `fork()`：将任务推入当前线程工作队列，异步执行
- `join()`：等待任务完成并获取结果

推荐 `left.fork() + right.compute()` 而非 `left.fork() + right.fork()` 的原因：
1. **减少工作队列操作**：当前线程直接执行 right，不需要入队再出队
2. **减少线程切换**：当前线程不会因队列为空而阻塞
3. **利用调用栈**：right.compute() 在当前线程栈上执行，数据局部性更好

两个都 fork 也能工作，但多了一次入队操作和潜在的窃取开销。

---

## Q3：ForkJoinTask 的 invoke() 和 fork() + join() 有什么区别？

**A**：
- `invoke()`：在当前线程同步执行任务，等价于 `task.fork(); task.join();`，但实现上直接在当前线程调用 `compute()`，避免入队开销
- `fork() + join()`：fork 将任务推入队列异步执行，join 等待结果

对于根任务，推荐用 `pool.invoke(task)` 直接提交执行；对于子任务，推荐 `left.fork() + right.compute()` 模式。

---
```java
// RecursiveTask：有返回值的递归任务
class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;
    private static final int THRESHOLD = 10000;

    SumTask(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();              // 异步执行左半
        return right.compute()    // 同步执行右半
            + left.join();        // 等待左半完成
    }
}

ForkJoinPool pool = new ForkJoinPool();
long total = pool.invoke(new SumTask(array, 0, array.length));
```

# ForkJoinTask fork/join

## Q1：fork() 和 join() 的执行原理是什么？

**A**：

- **fork()**：将子任务推入当前线程的工作队列（WorkQueue），不会立即执行，而是等线程调度执行或被其他线程窃取
- **join()**：等待子任务完成并获取结果。内部调用 `doJoin()`，如果任务未开始则尝试自己执行，如果正在执行则当前线程去执行其他任务（避免空等），直到目标任务完成

关键点：join 不会傻等，而是会**帮助执行队列中的其他任务**来利用等待时间。

---

## Q2：为什么推荐 left.fork() + right.compute() 而非两个都 fork？

**A**：
1. **当前线程不会空闲**：直接 compute right 任务，不需要等待
2. **减少队列操作**：right 不需要入队再出队，省去 push/pop 开销
3. **减少线程竞争**：少一个入队任务，被窃取的概率变化不大
4. **更好的缓存局部性**：compute 在当前调用栈上执行，共享父任务的数据

如果两个都 fork，当前线程的队列为空，要么从队列取回自己的任务（浪费入出队操作），要么被阻塞等待其他线程窃取执行。

---

## Q3：join() 方法会阻塞当前线程吗？它和 Future.get() 有什么区别？

**A**：  
**join() 会阻塞**，但有优化：阻塞期间当前线程会去执行工作队列中的其他任务，而非真正空转。但 join() **不支持中断**——如果任务抛异常，join 会将异常包装为 RuntimeException 重新抛出。
**与 Future.get() 的区别**：  

| 特性 | join() | get() |  
|------|--------|-------|  
| 中断支持 | 不支持 | 支持（InterruptedException） |  
| 超时 | 不支持 | 支持（get(timeout, unit)） |  
| 异常处理 | RuntimeException | ExecutionException |  
| 适用 | ForkJoin 内部 | Future 接口通用 |

---

# THRESHOLD 调优

## Q1：THRESHOLD 设太大会怎样？设太小会怎样？

**A**：THRESHOLD 是决定是否拆分的边界值，影响 ForkJoin 性能的关键参数：

| 阈值问题 | 现象 | 原因 |
|---------|------|------|
| **THRESHOLD 太大** | 任务粒度粗，CPU 利用率低，并行效果差 | 任务太少，负载均衡差 |
| THRESHOLD 太小 | fork/join 开销超过计算收益，整体变慢 | 子任务创建成本 > 并行收益 |

**理想场景**：每个子任务的 compute() 耗时在 **100μs ~ 10ms** 之间。

**经验公式**：
```java
// CPU 密集型：超分 4 倍
int threshold = array.length / (parallelism * 4);

// 保底值：不低于 1000
int threshold = Math.max(1_000, array.length / (parallelism * 4));
```

---

## Q2：如何确定一个合理的 THRESHOLD 值？

**A**：通过**实测对比**确定最简单有效：

```java
public class ThresholdBenchmark {
    public static void main(String[] args) {
        long[] array = new long[10_000_000];
        Arrays.fill(array, 1);

        int[] thresholds = {1_000, 5_000, 10_000, 50_000, 100_000, 500_000};

        for (int t : thresholds) {
            long start = System.nanoTime();
            ForkJoinPool pool = new ForkJoinPool();
            long sum = pool.invoke(new SumTask(array, 0, array.length, t));
            long ms = (System.nanoTime() - start) / 1_000_000;
            System.out.printf("threshold=%7d → %6d ms%n", t, ms);
        }
    }
}
```

典型结果（CPU 密集型，8 核）：
```
threshold=    1000 →  523 ms  ← fork 开销太大
threshold=    5000 →  201 ms  ← 开始有效果
threshold=   10000 →  145 ms  ← 接近最优
threshold=   50000 →  138 ms  ← 最优区间
threshold=  100000 →  156 ms  ← 任务太少
threshold=  500000 →  412 ms  ← 接近串行
```

---

## Q3：不同类型任务的 THRESHOLD 应该设多少？

**A**：THRESHOLD 的本质是"单次 compute 的目标耗时"：

| 任务类型 | 单次操作耗时 | 建议 THRESHOLD |
|---------|------------|----------------|
| 数组求和（long）| ~100ns/元素 | 50,000 ~ 100,000 |
| 数组排序 | O(n log n) | 1,000 ~ 10,000 |
| 文件哈希（MD5）| ~1ms/MB | 对应 ~10MB 数据量 |
| JSON 解析 | ~10μs/对象 | 500 ~ 2,000 |
| 数据库批量写入 | ~1ms/100条 | 1,000 ~ 5,000 |

**通用原则**：计算越快 → THRESHOLD 越大；计算越慢 → THRESHOLD 越小。

