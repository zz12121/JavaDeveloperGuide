
# CountedCompleter

## 核心结论

CountedCompleter 是 ForkJoinTask 的另一种形式，通过**待完成计数**（pending count）跟踪子任务完成情况。当计数归零时自动触发 `onCompletion()` 回调，适合**不确定子任务数量**或**需要完成通知**的场景。`addToPendingCount(n)` 注册 n 个待完成任务，每个子任务完成时 `tryComplete()` 递减计数。

## 深度解析

### 与 RecursiveTask 的区别

| 特性 | RecursiveTask | CountedCompleter |
|------|--------------|-----------------|
| 子任务数量 | 编译时确定 | 运行时动态确定 |
| 结果合并 | join + 手动合并 | onCompletion 回调 |
| 计数管理 | 无 | pendingCount 自动管理 |
| 完成触发 | 显式 join | 计数归零自动触发 |
| 适用场景 | 固定分治（二分） | 动态分治（搜索、MapReduce） |

### 核心方法

| 方法 | 作用 |
|------|------|
| `addToPendingCount(n)` | 增加 n 个待完成子任务 |
| `tryComplete()` | 完成当前任务，计数 -1，归零则触发 onCompletion |
| `propagateCompletion()` | 完成当前任务并传播到父任务 |
| `onCompletion(CountedCompleter)` | 所有子任务完成后的回调 |
| `getRawResult()` | 获取结果（默认返回 null） |

### 工作原理

```
Root (pendingCount = 3)
├── TaskA (pendingCount = 2)
│   ├── TaskA1 → tryComplete() → pendingCount 1
│   └── TaskA2 → tryComplete() → pendingCount 0 → onCompletion()
│   → propagateCompletion() → Root pendingCount 2
├── TaskB → tryComplete() → Root pendingCount 1
└── TaskC → tryComplete() → Root pendingCount 0 → onCompletion() → 完成！
```

### 实战示例：并行搜索

```java
class SearchTask extends CountedCompleter<String> {
    private final Node[] nodes;
    private final int lo, hi;
    private final String target;
    private volatile String result;

    SearchTask(CountedCompleter<?> parent, Node[] nodes,
               int lo, int hi, String target) {
        super(parent);
        this.nodes = nodes;
        this.lo = lo;
        this.hi = hi;
        this.target = target;
    }

    @Override
    public void compute() {
        if (hi - lo <= 10) {
            // 叶子任务：直接搜索
            for (int i = lo; i < hi; i++) {
                if (nodes[i].matches(target)) {
                    result = nodes[i].value;
                    // 找到后提前完成
                    super.quietlyCompleteRoot();
                    return;
                }
            }
        } else {
            // 拆分
            int mid = lo + (hi - lo) / 2;
            addToPendingCount(1);
            new SearchTask(this, nodes, lo, mid, target).fork();
            new SearchTask(this, nodes, mid, hi, target).fork();
        }
        // 通知当前任务已完成
        tryComplete();
    }

    @Override
    public String getRawResult() {
        return result;
    }
}
```

## 关联知识点

