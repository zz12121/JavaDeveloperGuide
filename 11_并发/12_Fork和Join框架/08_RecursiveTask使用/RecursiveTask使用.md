
# RecursiveTask使用

## 核心结论

RecursiveTask\<V\> 是 ForkJoinTask 的有返回值子类，用于**需要汇总结果的分治任务**。核心是实现 `compute()` 方法：判断任务是否足够小（阈值判断），小则直接计算，大则拆分 fork+join 合并结果。

## 深度解析

### 编写模板

```java
class MyTask extends RecursiveTask<V> {
    // 1. 定义任务数据和边界
    private final /* 数据 */ data;
    private static final int THRESHOLD = /* 拆分阈值 */;
    
    // 2. 构造函数传入子任务范围
    MyTask(/* 参数 */) { this.data = data; }
    
    // 3. 实现 compute()
    @Override
    protected V compute() {
        if (/* 任务足够小 */) {
            // 4. 直接计算
            return /* 直接结果 */;
        }
        // 5. 拆分为子任务
        MyTask left = new MyTask(/* 左半 */);
        MyTask right = new MyTask(/* 右半 */);
        left.fork();
        return right.compute() + left.join();  // 合并
    }
}
```

### 实战：归并排序

```java
class MergeSortTask extends RecursiveTask<long[]> {
    private final long[] arr;
    private final int lo, hi;
    private static final int THRESHOLD = 100;

    MergeSortTask(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    @Override
    protected long[] compute() {
        if (hi - lo <= THRESHOLD) {
            // 小数组直接用 Arrays.sort
            long[] sub = Arrays.copyOfRange(arr, lo, hi);
            Arrays.sort(sub);
            return sub;
        }
        int mid = lo + (hi - lo) / 2;
        MergeSortTask left = new MergeSortTask(arr, lo, mid);
        MergeSortTask right = new MergeSortTask(arr, mid, hi);
        left.fork();
        long[] rightResult = right.compute();
        long[] leftResult = left.join();
        return merge(leftResult, rightResult);
    }

    private long[] merge(long[] a, long[] b) {
        long[] result = new long[a.length + b.length];
        int i = 0, j = 0, k = 0;
        while (i < a.length && j < b.length)
            result[k++] = a[i] <= b[j] ? a[i++] : b[j++];
        while (i < a.length) result[k++] = a[i++];
        while (j < b.length) result[k++] = b[j++];
        return result;
    }
}

// 使用
long[] arr = {5, 3, 8, 1, 9, 2, 7, 4, 6};
long[] sorted = ForkJoinPool.commonPool()
    .invoke(new MergeSortTask(arr, 0, arr.length));
```

### 阈值选择建议

| 场景 | 建议阈值 | 原因 |
|------|---------|------|
| 简单累加 | 10,000~100,000 | 计算量小，拆分细一些 |
| 归并排序 | 100~1,000 | 排序本身 O(nlogn)，不需要太细 |
| 复杂计算 | 需要实测调优 | depends on per-element cost |

## 关联知识点
