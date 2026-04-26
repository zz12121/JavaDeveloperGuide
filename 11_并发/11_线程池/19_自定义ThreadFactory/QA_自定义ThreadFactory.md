
# 自定义ThreadFactory

## Q1：为什么要自定义 ThreadFactory？

**A**：默认 `Executors.defaultThreadFactory()` 存在三个问题：

1. **命名不可读**：生成 `pool-1-thread-1` 格式名称，多个线程池时无法区分
2. **无异常处理**：线程抛出未捕获异常时没有任何日志
3. **属性不可控**：无法设置守护线程、优先级等

生产环境必须自定义 ThreadFactory，至少实现线程命名和异常处理。

---

## Q2：ThreadFactory 接口有哪些常用实现？

**A**：

| 实现 | 来源 | 特点 |
|------|------|------|
| `DefaultThreadFactory` | JDK Executors | 默认实现，命名 pool-N-thread-M |
| `PrivilegedThreadFactory` | JDK Executors | 创建具有访问控制权限的线程 |
| `ThreadFactoryBuilder` | Guava | 链式 API，推荐使用 |
| `CustomizableThreadCreator` | Spring | Spring 环境下使用 |

---

## Q3：如何在生产环境中规范线程命名？

**A**：

1. **命名格式**：`{业务模块}-{功能}-%d`
   - 例：`order-processor-1`、`async-notify-3`
2. **使用 Guava ThreadFactoryBuilder**（推荐）：

```java
ThreadFactory factory = new ThreadFactoryBuilder()
    .setNameFormat("order-service-%d")
    .setDaemon(false)
    .setUncaughtExceptionHandler((t, e) -> 
        log.error("线程[{}]异常", t.getName(), e))
    .build();
```

3. **排查便利**：命名后可通过 `jstack`、日志、APM 快速定位线程来源

> **代码示例：生产级 ThreadFactory**

```java
public class ProductionThreadFactory implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setUncaughtExceptionHandler((thread, ex) ->
            log.error("Uncaught in [{}]: {}", thread.getName(), ex.getMessage(), ex));
        return t;
    }
}
```

