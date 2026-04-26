
# 线程池关闭

## Q1：shutdown 和 shutdownNow 的区别？

**A**：

| 维度 | shutdown | shutdownNow |
|------|---------|-------------|
| 新任务 | 拒绝 | 拒绝 |
| 队列任务 | 继续执行 | 丢弃（返回列表） |
| 执行中任务 | 继续执行 | 中断 |
| 推荐 | ✅ 优先使用 | 兜底使用 |

---

## Q2：如何优雅关闭线程池？

**A**：

```java
pool.shutdown(); // 停止接受新任务
if (!pool.awaitTermination(60, SECONDS)) { // 等待60秒
    pool.shutdownNow(); // 超时强制关闭
    pool.awaitTermination(60, SECONDS); // 再等一会
}
```

---

## Q3：不关闭线程池有什么后果？

**A**：

1. **JVM 无法退出**：线程池中的线程是非守护线程，JVM 会等待它们终止
2. **资源泄漏**：线程占用内存和 CPU
3. **任务丢失**：队列中未执行的任务不会被处理

在 Spring 中，建议用 `@PreDestroy` 关闭线程池。

