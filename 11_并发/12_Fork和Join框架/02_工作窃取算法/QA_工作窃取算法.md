
# 工作窃取算法

## Q1：什么是工作窃取算法？它是如何工作的？

**A**：工作窃取（Work Stealing）是 ForkJoinPool 的调度策略。

**工作机制**：
1. 每个工作线程有自己的**双端队列**（WorkQueue）
2. 线程从自己队列**头部（LIFO）**取任务执行
3. 执行过程中 fork 的子任务也放入自己队列头部
4. 当线程空闲时，从**其他线程队列尾部（FIFO）**"偷"任务执行

**设计优势**：头部 LIFO 利用缓存局部性（子任务数据热），尾部 FIFO 让窃取线程拿到较大的任务便于拆分，头尾分离天然减少竞争。

---

## Q2：工作窃取算法中为什么从头部取、从尾部偷？

**A**：
- **本线程从头部取（LIFO/栈式）**：fork 产生的子任务推入头部，LIFO 取最近的任务，数据大概率还在 CPU 缓存中（空间局部性好）
- **其他线程从尾部偷（FIFO/队列式）**：尾部是较早上来尚未拆分的"大任务"，适合被偷走后进一步 fork 拆分

这种头尾分离使得**生产者和消费者操作不同的端**，大幅减少 CAS 竞争。

---

## Q3：工作窃取有没有缺点？

**A**：
1. **窃取开销**：每次窃取需要 CAS 操作，任务粒度过小时，窃取开销可能超过并行收益
2. **双端队列开销**：每个线程维护独立队列，比共享队列占用更多内存
3. **不适合所有场景**：I/O 密集型或任务间有复杂依赖时，窃取效果不好
4. **线程数选择**：并行度过高会导致窃取竞争增加，一般设为 CPU 核心数 - 1

> **代码示例：使用 ForkJoinTask 观察工作窃取效果**

```java
// 递归求和：大任务会 fork 拆分为子任务，空闲线程窃取执行
class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int lo, hi;
    private static final int THRESHOLD = 10_000;

    SumTask(long[] array, int lo, int hi) {
        this.array = array; this.lo = lo; this.hi = hi;
    }

    @Override
    protected Long compute() {
        if (hi - lo < THRESHOLD) {
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += array[i];
            return sum; // 小任务直接计算
        }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(array, lo, mid);
        SumTask right = new SumTask(array, mid, hi);
        left.fork();        // 子任务入队
        return right.compute() + left.join(); // 本线程计算右半，join 等待左半（可能被窃取）
    }
}
```
