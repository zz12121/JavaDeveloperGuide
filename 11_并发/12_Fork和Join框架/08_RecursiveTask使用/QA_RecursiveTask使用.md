
# RecursiveTask使用

## Q1：如何正确编写一个 RecursiveTask？

**A**：标准模板：
1. 继承 `RecursiveTask<V>`，V 是返回类型
2. 定义数据范围和拆分阈值
3. 实现 `compute()`：
   - 如果任务足够小（达到阈值），**直接计算返回**
   - 否则**拆分**为左右子任务，`left.fork()` 异步执行左半，`right.compute()` 当前线程执行右半，`left.join()` 等待合并

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int lo, hi;
    
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

---

## Q2：拆分阈值怎么选择？是不是越小越好？

**A**：**不是越小越好**。阈值过小会导致：
1. 任务数量暴增，队列和调度开销增大
2. fork/join 本身有 CAS 和上下文切换开销
3. 可能比单线程串行还慢

阈值过大则并行度不够，无法充分利用 CPU。一般建议：
- 简单操作（如求和）：10,000~100,000
- 复杂操作（如排序）：100~1,000
- **最佳方式**：实际基准测试（JMH）调优

---

## Q3：RecursiveTask 的 compute() 可以多次 fork 吗？

**A**：可以，但不推荐 fork 两个。正确做法是 `left.fork() + right.compute()`，只 fork 一个子任务，另一个直接在当前线程 compute。如果拆分为 3 个或更多子任务，可以用 `invokeAll(task1, task2, task3)`，它会合理分配执行。


