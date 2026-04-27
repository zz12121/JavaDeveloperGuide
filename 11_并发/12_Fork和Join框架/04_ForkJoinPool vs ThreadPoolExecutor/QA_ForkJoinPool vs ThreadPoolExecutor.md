
# ForkJoinPool vs ThreadPoolExecutor

## Q1：ForkJoinPool 和 ThreadPoolExecutor 的核心区别是什么？

**A**：核心区别在于**任务模型和调度策略**：

- **ThreadPoolExecutor**：处理独立任务，所有线程从**共享阻塞队列**竞争获取任务，适合通用并发（Web 请求、定时任务）
- **ForkJoinPool**：处理可递归拆分的父子任务，每个线程有**独立双端队列**，通过**工作窃取**调度，适合分治计算（排序、求和、MapReduce）

简单说：TPE 是"工人排队领活"，FJP 是"工人的活自己会生小孩，空闲的工人去偷别人的小孩做"。

---

## Q2：什么场景下应该用 ForkJoinPool 而不是 ThreadPoolExecutor？

**A**：
- ✅ **用 ForkJoinPool**：大任务可拆分为相似子任务、CPU 密集型计算、递归并行处理（归并排序、目录遍历、矩阵运算）、使用 parallel stream
- ❌ **不适合 ForkJoinPool**：I/O 密集任务、任务间有复杂依赖、任务不可拆分

关键判断标准：任务是否体现**分治（Divide and Conquer）**特征。

---

## Q3：能在 ForkJoinPool 中执行 I/O 阻塞任务吗？

**A**：**强烈不建议**。ForkJoinPool 的工作线程数量有限（默认 CPU 核数 - 1），如果任务中包含阻塞 I/O（如 sleep、网络请求、数据库查询），线程被阻塞后：
1. 无法处理自己队列中的其他任务
2. 其他线程窃取时少了工作来源
3. 整体吞吐量急剧下降

I/O 密集任务应该用 ThreadPoolExecutor + 合适的队列和线程数。



> **代码示例：同一任务的两种实现对比**

```java
// 方式1：ThreadPoolExecutor（独立任务）
ExecutorService tpe = Executors.newFixedThreadPool(4);
List<Future<Integer>> results = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    final int n = i;
    results.add(tpe.submit(() -> heavyCalc(n))); // 每个任务独立
}

// 方式2：ForkJoinPool（可拆分任务）
ForkJoinPool fjp = new ForkJoinPool(4);
int total = fjp.invoke(new RecursiveTask<Integer>() {
    @Override
    protected Integer compute() {
        if (end - start < THRESHOLD) return sequentialCalc();
        int mid = (start + end) / 2;
        var left = new SubTask(start, mid);
        var right = new SubTask(mid, end);
        left.fork();
        return right.compute() + left.join(); // 父子任务有依赖
    }
});
```
