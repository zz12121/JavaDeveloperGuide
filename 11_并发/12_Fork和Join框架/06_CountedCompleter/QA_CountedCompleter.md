
# CountedCompleter

## Q1：CountedCompleter 和 RecursiveTask 的核心区别是什么？什么时候用 CountedCompleter？

**A**：核心区别在于**结果收集方式**：

- **RecursiveTask**：需要显式 `fork()` + `join()` 收集子任务结果
- **CountedCompleter**：不需要 join，通过 pending count 自动触发 `onCompletion()` 回调

```
RecursiveTask:
  fork(left) → fork(right) → join(left) + join(right) → 合并结果
      ↑ 需要手动等待每个子任务完成

CountedCompleter:
  addToPendingCount + fork() → 子任务完成后自动 tryComplete()
      → pending == 0 → 自动触发 onCompletion()
      ↑ 不需要手动 join
```

**用 CountedCompleter 当**：
- 只需执行动作，不需要收集每个子任务的具体结果
- "扇出-合并"模式：遍历目录树、爬虫抓取、批量写入
- 父任务需要等待所有子任务完成后才能触发后续处理

**用 RecursiveTask 当**：
- 子任务结果需要汇总（求和、排序、找最大/最小值）
- 典型的分治递归计算

---

## Q2：CountedCompleter 的 pending count 机制是怎么工作的？

**A**：pending count 是 CountedCompleter 的核心，通过计数器管理子任务的生命周期：

```
初始：pending = 0
  ↓ addToPendingCount(N) + fork() × N个子任务
pending = N（N个子任务待完成）
  ↓ 每个子任务完成时调用 tryComplete()
pending = N - 1 → N - 2 → ... → 0
  ↓ pending == 0
自动触发 onCompletion() 回调
```

**tryComplete() 源码逻辑**：

```java
public final void tryComplete() {
    CountedCompleter<?> a = this;
    for (int c;;) {
        if ((c = a.pending) == 0) {          // 计数器为0
            a.onCompletion(a);               // 触发回调
            if ((a = a.completer) == null) { // 向上传递给父任务
                return;
            }
        } else if (a.compareAndSetPendingCount(c, c - 1)) {
            // 递减成功，还有子任务未完成
            return;
        }
    }
}
```

**注意事项**：
- `addToPendingCount()` 必须在 `fork()` 之前调用（原子性保证）
- `tryComplete()` 可以调用多次，只有 pending == 0 时才真正触发回调
- 可以用 `propagateCompletion()` 无条件向上传递完成信号

---

## Q3：CountedCompleter 如何处理异常？

**A**：CountedCompleter 的异常机制与 RecursiveTask 相同，但可以覆盖 `onExceptionalCompletion()` 做专门的异常处理：

```java
public class SafeFileCounter extends CountedCompleter<Void> {
    private final Map<String, Long> result;

    @Override
    public void compute() {
        try {
            // 正常处理逻辑
            for (File child : dir.listFiles()) {
                if (child.isDirectory()) {
                    addToPendingCount(1);
                    new SafeFileCounter(this, child, result).fork();
                } else {
                    // ...
                }
            }
            tryComplete();
        } catch (Throwable t) {
            // 异常时主动触发完成，避免永远悬挂
            propagateCompletion();
        }
    }

    @Override
    public void onExceptionalCompletion(Throwable ex, CountedCompleter<?> caller) {
        // 异常完成时的回调
        System.err.println("处理异常: " + dir + ", " + ex.getMessage());
        // 可在此记录日志、发送告警
    }
}
```

关键点：**在 catch 中调用 `propagateCompletion()`**，否则 pending count 不会归零，父任务永远等待。

---

## Q4：目录树遍历用 RecursiveTask 还是 CountedCompleter？

**A**：**推荐 CountedCompleter**，原因如下：

**RecursiveTask 方式**（需要手动 join）：
```java
class DirSizeTask extends RecursiveTask<Long> {
    @Override
    protected Long compute() {
        File[] children = dir.listFiles();
        long sum = 0;
        List<DirSizeTask> subTasks = new ArrayList<>();

        for (File child : children) {
            if (child.isDirectory()) {
                DirSizeTask sub = new DirSizeTask(child);
                subTasks.add(sub);
                sub.fork();
            } else {
                sum += child.length();
            }
        }

        // 必须手动 join 所有子任务
        for (DirSizeTask sub : subTasks) {
            sum += sub.join();
        }
        return sum;
    }
}
```

**CountedCompleter 方式**（自动合并）：
```java
class DirSizeTask2 extends CountedCompleter<Long> {
    private final AtomicLong result; // 共享结果

    @Override
    public void compute() {
        File[] children = dir.listFiles();
        for (File child : children) {
            if (child.isDirectory()) {
                addToPendingCount(1);
                new DirSizeTask2(this, result).fork();
            } else {
                result.addAndGet(child.length());
            }
        }
        tryComplete(); // 自动等待所有子任务
    }

    @Override
    public void onCompletion(CountedCompleter<?> caller) {
        // 完成后可执行额外逻辑
    }
}
```

**对比**：CountedCompleter 更简洁，不需要手动遍历 fork 列表去 join，尤其适合目录层级深的场景。

---

## Q5：CountedCompleter 的 addToPendingCount 为什么必须在 fork 之前调用？

**A**：`addToPendingCount` 不是线程安全的 CAS 操作，它直接设置 `pending = pending + delta`，不是原子递增。如果在 fork 之后调用，可能出现竞态：

```
线程A: fork() → 线程B已窃取任务 → 子任务已执行完调用 tryComplete()
      → pending 还没增加就减为0 → onCompletion 提前触发！
```

正确顺序：
```java
addToPendingCount(1);    // ✅ 先增加计数
new SubTask(this).fork(); // 再 fork
```

如果需要动态 fork 多个子任务：
```java
for (File child : children) {
    if (child.isDirectory()) {
        addToPendingCount(1);          // 每个子任务前都先增加计数
        new SubTask(this, child).fork();
    }
}
tryComplete();                          // 最后尝试完成
```

---

```java
// 完整示例：并行爬取网页
public class WebCrawler extends CountedCompleter<Void> {
    private final String url;
    private final Set<String> visited;
    private final ConcurrentMap<String, Integer> pageSizes;
    private static final int MAX_DEPTH = 3;

    public WebCrawler(CountedCompleter<?> parent, String url,
                      Set<String> visited, ConcurrentMap<String, Integer> pageSizes) {
        super(parent);
        this.url = url;
        this.visited = visited;
        this.pageSizes = pageSizes;
    }

    public WebCrawler(String url, Set<String> visited,
                      ConcurrentMap<String, Integer> pageSizes) {
        this(null, url, visited, pageSizes);
    }

    @Override
    public void compute() {
        if (!visited.add(url)) {
            tryComplete(); // 已访问过，直接完成
            return;
        }

        List<String> links = fetchAndParse(url); // 获取页面 + 解析链接
        if (links == null || links.isEmpty()) {
            tryComplete();
            return;
        }

        for (String link : links) {
            addToPendingCount(1);
            new WebCrawler(this, link, visited, pageSizes).fork();
        }
        tryComplete();
    }

    private List<String> fetchAndParse(String url) {
        // 实际实现：HTTP 请求 + HTML 解析
        // 返回页面中的链接列表
        return List.of();
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        Set<String> visited = ConcurrentHashMap.newKeySet();
        ConcurrentMap<String, Integer> pageSizes = new ConcurrentHashMap<>();
        pool.invoke(new WebCrawler("https://example.com", visited, pageSizes));
        System.out.println("爬取页面数: " + visited.size());
    }
}
```
