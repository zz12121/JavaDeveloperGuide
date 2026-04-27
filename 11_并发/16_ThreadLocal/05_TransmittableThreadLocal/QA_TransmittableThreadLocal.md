
# TransmittableThreadLocal

## Q1：TransmittableThreadLocal 是什么？解决什么问题？

**A**：

TTL 是阿里开源的线程本地变量传递框架，解决 `InheritableThreadLocal` 在**线程池场景下值无法正确传递**的问题。

核心思路：在**提交任务时捕获**（capture）当前线程的 TTL 值，在任务**执行前恢复**（replay）到子线程，执行后**还原**（restore）子线程的原值。这样每次提交任务都会重新传递最新的值。

---

## Q2：TTL 的 capture/replay/restore 机制是怎样的？

**A**：

1. **capture()**：在任务提交时，遍历所有注册的 TTL，将当前线程的值存入快照 Map
2. **wrap(Runnable)**：创建 `TtlRunnable`，内部持有快照引用
3. **replay(captured)**：在子线程执行 `run()` 前，用快照覆盖子线程的 TTL 值，同时备份子线程原值
4. **restore(backup)**：`run()` 执行完毕后，恢复子线程原来的 TTL 值

```java
// 简化原理
class TtlRunnable implements Runnable {
    Object captured;  // 提交时的快照
    
    public void run() {
        Object backup = replay(captured); // 设置快照，备份原值
        try {
            runnable.run();               // 执行实际任务
        } finally {
            restore(backup);              // 恢复原值
        }
    }
}
```

---

## Q3：TTL 有哪些使用方式？

**A**：

1. **装饰线程池**（推荐）：
   ```java
   ExecutorService pool = TtlExecutors.getTtlExecutorService(
       Executors.newFixedThreadPool(4)
   );
   ```

2. **手动包装 Runnable**：
   ```java
   Runnable ttlRunnable = TtlRunnable.get(runnable);
   pool.execute(ttlRunnable);
   ```

3. **Java Agent 无侵入**：启动参数 `-javaagent:ttl.jar`，自动增强所有线程池，无需改代码

---

## Q4：TTL 和 InheritableThreadLocal 有什么区别？

**A**：

| 维度 | InheritableThreadLocal | TTL |
|------|----------------------|-----|
| 传递时机 | `new Thread()` 时拷贝一次 | 每次提交任务时捕获 |
| 线程池 | ❌ 复用线程值不更新 | ✅ 每次传递最新值 |
| 值恢复 | 不支持 | 执行完恢复原值 |
| 依赖 | JDK 内置 | 需引入 alibaba TTL |
| 性能 | 无额外开销 | capture/replay 有少量开销 |

