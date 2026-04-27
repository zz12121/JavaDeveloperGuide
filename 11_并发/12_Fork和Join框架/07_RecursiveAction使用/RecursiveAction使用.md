
# RecursiveAction使用

## 核心结论

RecursiveAction 是 ForkJoinTask 的**无返回值**子类，用于**只需执行动作、不需要返回结果**的分治任务。`compute()` 返回 void，用 `invokeAll()` 并行执行多个子任务。

## 深度解析

### 与 RecursiveTask 的区别

```java
// RecursiveTask<V>：有返回值
class SumTask extends RecursiveTask<Long> {
    protected Long compute() {
        return left.join() + right.compute();  // 合并结果
    }
}

// RecursiveAction：无返回值
class DoubleTask extends RecursiveAction {
    protected void compute() {
        invokeAll(leftTask, rightTask);  // 执行，不合并
    }
}
```

### 典型场景

- 遍历文件目录并处理文件
- 批量更新数组元素
- 并行写入/发送数据
- 树结构遍历

### 实战：并行遍历文件目录

```java
class FileProcessTask extends RecursiveAction {
    private final File dir;
    private final Consumer<File> processor;

    FileProcessTask(File dir, Consumer<File> processor) {
        this.dir = dir;
        this.processor = processor;
    }

    @Override
    protected void compute() {
        File[] files = dir.listFiles();
        if (files == null) return;
        
        List<FileProcessTask> subtasks = new ArrayList<>();
        for (File file : files) {
            if (file.isDirectory()) {
                subtasks.add(new FileProcessTask(file, processor));
            } else {
                processor.accept(file);  // 处理文件
            }
        }
        if (!subtasks.isEmpty()) {
            invokeAll(subtasks);  // 并行处理子目录
        }
    }
}

// 使用
ForkJoinPool pool = new ForkJoinPool();
pool.invoke(new FileProcessTask(
    new File("/path/to/dir"),
    file -> System.out.println(file.getName())
));
```

### 实战：并行数组操作

```java
class ArrayMultiplyTask extends RecursiveAction {
    private final double[] arr;
    private final int lo, hi;
    private final double factor;
    private static final int THRESHOLD = 50_000;

    ArrayMultiplyTask(double[] arr, int lo, int hi, double factor) {
        this.arr = arr; this.lo = lo; this.hi = hi; this.factor = factor;
    }

    @Override
    protected void compute() {
        if (hi - lo <= THRESHOLD) {
            for (int i = lo; i < hi; i++) arr[i] *= factor;
        } else {
            int mid = lo + (hi - lo) / 2;
            invokeAll(
                new ArrayMultiplyTask(arr, lo, mid, factor),
                new ArrayMultiplyTask(arr, mid, hi, factor)
            );
        }
    }
}
```

## 关联知识点

