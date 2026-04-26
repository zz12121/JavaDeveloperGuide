
# newSingleThreadExecutor

## 核心结论

单线程线程池：只有一个核心线程，保证任务按提交顺序执行。使用无界 LinkedBlockingQueue。**与 FixedThreadPool(1) 的区别在于返回的 ExecutorService 包装类不同**。

## 深度解析

### 创建参数

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(
        new ThreadPoolExecutor(
            1, 1,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>()
        )
    );
}
```

### vs FixedThreadPool(1)

```java
// newSingleThreadExecutor
FinalizableDelegatedExecutorService → 无法强转调用 setCorePoolSize 等方法

// newFixedThreadPool(1)
ThreadPoolExecutor → 可以强转调用 setCorePoolSize 修改线程数
```

包装后暴露的是 ExecutorService 接口，**隐藏了 ThreadPoolExecutor 的特有方法**，防止运行时被修改参数。

### 特性

| 维度 | 说明 |
|------|------|
| 线程数 | 始终 1 |
| 任务顺序 | FIFO（按提交顺序执行） |
| 队列 | 无界 LinkedBlockingQueue |
| 异常 | 一个任务异常不影响后续任务 |
| OOM 风险 | ⚠️ 有（无界队列） |

### 适用场景

- 需要顺序执行的任务
- 单点写入（如日志顺序写入）
- 不需要并行的场景

## 易错点与踩坑

### 1. 与 newFixedThreadPool(1) 的区别（面试常问）
```java
// newSingleThreadExecutor → FinalizableDelegatedExecutorService 包装
// 返回 ExecutorService 接口，无法强转为 ThreadPoolExecutor
ExecutorService e1 = Executors.newSingleThreadExecutor();
// ((ThreadPoolExecutor) e1).setCorePoolSize(5); // ClassCastException!

// newFixedThreadPool(1) → 直接返回 ThreadPoolExecutor
ExecutorService e2 = Executors.newFixedThreadPool(1);
// ((ThreadPoolExecutor) e2).setCorePoolSize(5); // 可以！运行时修改了线程数
```

### 2. 无界队列同样的 OOM 风险
- 使用 `LinkedBlockingQueue()`（无界），与 FixedThreadPool 相同
- 单线程处理速度有限，任务堆积 → OOM

### 3. 异常不会"中断"线程池
```java
// 一个任务异常不会导致线程死亡
// 线程池会继续从队列取下一个任务
// 但异常信息可能丢失（execute 模式下）
```

### 4. FinalizableDelegatedExecutorService 的 finalize 陷阱
- 包装类在 GC 时会调用 `shutdown()`
- 如果线程池对象被回收，会自动关闭线程池
- **生产中不应该依赖 GC 来管理线程池生命周期**
## 关联知识点

