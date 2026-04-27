# ForkJoin 异常处理

## 核心结论

ForkJoinTask 的异常处理机制与普通 Future 不同：`join()` 将子任务的异常**重新包装为 RuntimeException 抛出**（不是 ExecutionException），且 **invokeAll 中的某个子任务异常会导致其他子任务被取消**。必须主动通过 `getException()` 获取原始异常，而非依赖 catch 捕获。

## 深度解析

### ForkJoinTask 状态机

ForkJoinTask 通过 volatile int status 字段维护状态机：

```
fork() 产生子任务
    ↓
status = 0 (进行中)
    ↓
┌─────────────────────────────────────┐
│ 正常完成    → NORMAL (0)             │
│ 主动取消    → CANCELLED (-1)         │
│ 异常退出    → EXCEPTIONAL (-1)        │
│ 有线程等待  → SIGNAL (-2)            │
└─────────────────────────────────────┘
    ↓
join() / get() 读取状态，返回结果或抛异常
```

### join() vs get() 异常类型对比

| 特性 | join() | get() |
|------|--------|-------|
| 异常包装 | RuntimeException | ExecutionException |
| 中断支持 | ❌ 不支持 | ✅ 支持（抛出 InterruptedException）|
| 超时支持 | ❌ 不支持 | ✅ 支持（抛出 TimeoutException）|
| 推荐场景 | ForkJoin 内部子任务 | 外部调用者获取结果 |

**为什么 join() 用 RuntimeException 而不是 ExecutionException？**

ForkJoinPool 的设计哲学是**简化递归调用的异常处理**。在递归分治中，调用栈层级深，如果每个子任务都用 get()，需要层层 throws 或 try-catch。用 RuntimeException 可以一路抛出到顶层，由顶层统一处理。

### 异常传播机制

```java
public class ExceptionTask extends RecursiveTask<Integer> {
    @Override
    protected Integer compute() {
        if (someCondition) {
            throw new IllegalArgumentException("参数错误"); // 不是 checked，无需 declare
        }
        // ...
        return result;
    }

    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        ExceptionTask task = new ExceptionTask();
        pool.execute(task);
        try {
            task.join();  // 抛出 RuntimeException("java.lang.IllegalArgumentException: 参数错误")
        } catch (RuntimeException e) {
            // ❗ 这里拿到的是 RuntimeException，不是 IllegalArgumentException
            // 需调用 getException() 获取原始异常
            Throwable cause = task.getException();
            System.out.println("原始异常: " + cause.getClass().getName());
            System.out.println("消息: " + cause.getMessage());
        }
    }
}
```

### invokeAll 的异常处理特性

invokeAll 的异常处理有一个关键特性：**某个子任务异常会导致其他子任务被取消**。

```java
@Override
protected Integer compute() {
    if (end - start <= THRESHOLD) {
        return sequentialCompute();
    }
    try {
        // 子任务 t1 或 t2 抛异常，另一个会被 cancel
        invokeAll(t1, t2);
        return t1.join() + t2.join();
    } catch (Throwable e) {
        // 只会捕获到其中一个异常
        return handleException(e);
    }
}
```

**原理**：invokeAll 内部调用 `ForkJoinTask.cancel()`，取消状态为 NORMAL 以外的所有任务。

### 最佳实践：统一异常处理模板

```java
public class SafeRecursiveTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;
    private static final int THRESHOLD = 10_000;

    SafeRecursiveTask(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            try {
                return sequentialSum();
            } catch (Exception e) {
                throw new RuntimeException("计算失败: start=" + start, e);
            }
        }

        int mid = start + (end - start) / 2;
        SafeRecursiveTask left = new SafeRecursiveTask(array, start, mid);
        SafeRecursiveTask right = new SafeRecursiveTask(array, mid, end);

        // 捕获左侧异常，不影响右侧继续执行
        try {
            left.fork();
            Long rightResult = right.compute();
            Long leftResult = left.join();
            return leftResult + rightResult;
        } catch (RuntimeException e) {
            // 记录异常，但继续抛出
            System.err.println("子任务异常: " + e.getMessage());
            throw e;
        }
    }

    private Long sequentialSum() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += array[i];
        }
        return sum;
    }
}
```

### getException() 与 isCompletedAbnormally()

```java
// 主动获取异常（不抛异常）
ForkJoinTask<?> task = someTask;
if (task.isCompletedAbnormally()) {
    Throwable ex = task.getException();
    // 单独处理异常，不影响主流程
    logger.error("任务异常", ex);
}
```

### checked exception 处理

ForkJoinTask 不直接传播 checked exception，需要手动包装：

```java
@Override
protected Integer compute() throws IOException {
    try {
        return parseFile();
    } catch (IOException e) {
        throw new RuntimeException(e); // checked → unchecked 包装
    }
}

// 调用处
try {
    task.join();
} catch (RuntimeException e) {
    if (e.getCause() instanceof IOException) {
        // 还原处理
    }
}
```

## 关联知识点

- ForkJoinTask.invokeAll：异常传播导致子任务取消
- ForkJoinTask.join()：RuntimeException 包装
- ForkJoinTask.get()：ExecutionException 包装，支持中断和超时
