
# CountedCompleter

## Q1：什么是 CountedCompleter？它和 RecursiveTask 有什么区别？

**A**：CountedCompleter 是 ForkJoinTask 的一种形式，通过 **pendingCount**（待完成计数）来跟踪子任务完成情况，当计数归零时自动触发 `onCompletion()` 回调。

**核心区别**：
- **RecursiveTask**：子任务数量固定，通过 fork/join 手动合并结果
- **CountedCompleter**：子任务数量可动态变化，通过 pendingCount 自动管理，计数归零触发回调

CountedCompleter 适合**搜索、MapReduce** 等不确定子任务数量的场景。

---

## Q2：CountedCompleter 的 pendingCount 是怎么工作的？

**A**：
1. 父任务调用 `addToPendingCount(n)` 注册 n 个待完成子任务
2. 每个子任务完成后调用 `tryComplete()`，内部将 pendingCount - 1
3. 如果 pendingCount 归零，自动触发 `onCompletion()` 回调
4. `onCompletion()` 中通过 `propagateCompletion()` 传播到父任务

流程：父注册 → 子执行 → 子 tryComplete → 计数减 1 → 归零触发回调 → 传播到父 → 父计数减 1 → ... → 根任务归零 → 整体完成

---

## Q3：什么场景下该用 CountedCompleter 而不是 RecursiveTask？

**A**：
- **用 CountedCompleter**：子任务数量运行时才确定（如搜索树、动态拆分）、需要在全部子任务完成后触发统一操作、MapReduce 风格的汇总
- **用 RecursiveTask**：子任务数量固定（如二分拆分）、每个子任务返回结果需要显式合并

简单判断：如果 join + 手动合并能满足需求就用 RecursiveTask；如果子任务数量动态或需要完成回调就用 CountedCompleter。

---
```java
// CountedCompleter：完成计数后触发回调
class MapReduceTask extends CountedCompleter<Void> {
    private final List<String> data;

    MapReduceTask(List<String> data) { this.data = data; }

    @Override
    public void compute() {
        List<List<String>> parts = partition(data);
        addToPendingCount(parts.size() - 1);  // 设置待完成计数

        for (List<String> part : parts.subList(1, parts.size())) {
            new MapReduceTask(part).fork();    // fork 子任务
        }
        process(parts.get(0));                 // 当前线程处理第一份

        tryComplete();  // 递减计数，为 0 时调用 onCompletion
    }

    @Override
    public void onCompletion(CountedCompleter<?> caller) {
        // 所有子任务完成后触发
        mergeResults();
    }
}
```


