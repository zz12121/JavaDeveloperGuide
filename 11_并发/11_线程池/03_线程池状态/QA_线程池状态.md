
# 线程池状态

## Q1：线程池有哪些状态？

**A**：

| 状态 | 说明 |
|------|------|
| RUNNING | 接受新任务，处理队列 |
| SHUTDOWN | 不接受新任务，处理队列剩余 |
| STOP | 不接受新任务，不处理队列，中断进行中任务 |
| TIDYING | 所有任务完成，线程数为0 |
| TERMINATED | terminated() 完成，线程池终止 |

---

## Q2：ctl 是怎么设计的？

**A**：用 AtomicInteger 的高 3 位存状态，低 29 位存线程数：

```java
ctl = (运行状态 << 29) | 线程数

runStateOf(ctl)  → 高3位（状态）
workerCountOf(ctl) → 低29位（线程数）
isRunning(ctl)   → 状态 < SHUTDOWN（即 RUNNING）
```

CAS 操作 ctl 可以同时更新状态和线程数，保证原子性。

---

## Q3：shutdown 和 shutdownNow 的区别？

**A**：

- **shutdown()**：温和关闭，不接受新任务，但会处理完队列中的任务
- **shutdownNow()**：暴力关闭，不接受新任务，不处理队列，中断正在执行的任务，返回未执行的任务列表

