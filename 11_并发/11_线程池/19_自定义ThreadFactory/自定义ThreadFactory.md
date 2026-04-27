
# 自定义ThreadFactory

## 核心结论

自定义 ThreadFactory 是线程池最佳实践的核心环节，用于**规范化线程命名、设置异常处理器、控制线程属性**。默认的 ThreadFactory 创建的线程命名为 `pool-N-thread-M`，排查问题时无法区分线程来源，且没有异常处理机制。

## 深度解析

### 默认 ThreadFactory 的问题

```java
// Executors.defaultThreadFactory() 的实现
public Thread newThread(Runnable r) {
    Thread t = new Thread(group, r, "pool-N-thread-M", 0);
    if (t.isDaemon()) t.setDaemon(false);    // 默认非守护
    if (t.getPriority() != Thread.NORM_PRIORITY) t.setPriority(Thread.NORM_PRIORITY);
    return t;  // 没有 UncaughtExceptionHandler！
}
```

问题：
1. **命名不可读**：`pool-1-thread-1` 无法区分业务
2. **无异常处理**：线程抛出未捕获异常时无日志记录
3. **属性不可控**：无法设置守护线程、优先级等

### 自定义实现方式

#### 方式一：实现 ThreadFactory 接口

```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
    private final boolean isDaemon;

    public NamedThreadFactory(String namePrefix, boolean isDaemon) {
        this.namePrefix = namePrefix;
        this.isDaemon = isDaemon;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, namePrefix + "-" + threadNumber.getAndIncrement());
        t.setDaemon(isDaemon);
        t.setUncaughtExceptionHandler((thread, ex) ->
            log.error("线程[{}]发生异常: {}", thread.getName(), ex.getMessage(), ex));
        return t;
    }
}

// 使用
new NamedThreadFactory("order-service", false)
```

#### 方式二：Guava ThreadFactoryBuilder（推荐）

```java
import com.google.common.util.concurrent.ThreadFactoryBuilder;

ThreadFactory factory = new ThreadFactoryBuilder()
    .setNameFormat("order-service-%d")       // 线程命名
    .setDaemon(false)                          // 非守护线程
    .setPriority(Thread.NORM_PRIORITY - 1)     // 优先级
    .setUncaughtExceptionHandler((t, e) ->
        log.error("线程[{}]异常", t.getName(), e))
    .build();
```

#### 方式三：Spring 自定义

```java
// 使用 CustomizableThreadCreator（Spring 提供）
CustomizableThreadCreator factory = new CustomizableThreadCreator("async-");
factory.setDaemon(false);
factory.setThreadPriority(Thread.NORM_PRIORITY);
```

### 生产环境命名规范

| 线程池用途 | 命名示例 |
|-----------|---------|
| 订单处理 | `order-processor-%d` |
| 异步通知 | `async-notify-%d` |
| 定时任务 | `scheduled-task-%d` |
| HTTP 请求 | `http-client-%d` |
| 日志写入 | `log-writer-%d` |

命名格式：`{业务模块}-{功能}-%d`，方便在日志、jstack、监控中快速定位问题。

## 易错点与踩坑

- ❌ 使用默认 ThreadFactory，线程异常时无任何日志
- ✅ 必须设置 UncaughtExceptionHandler
- ❌ 线程池设为守护线程，JVM 退出时任务可能丢失
- ✅ 一般业务线程池设为非守护线程
- ❌ ThreadFactory 中的异常处理器执行耗时操作
- ✅ 异常处理器应尽量轻量，只记录日志

## 代码示例

```java
// 生产级 ThreadFactory 完整示例
public class ProductionThreadFactory implements ThreadFactory {
    private static final Logger log = LoggerFactory.getLogger(ProductionThreadFactory.class);
    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;
    private final boolean daemon;

    public ProductionThreadFactory(String prefix) {
        this(prefix, false);
    }

    public ProductionThreadFactory(String prefix, boolean daemon) {
        this.prefix = prefix;
        this.daemon = daemon;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setDaemon(daemon);
        t.setPriority(Thread.NORM_PRIORITY);
        t.setUncaughtExceptionHandler((thread, ex) ->
            log.error("Uncaught exception in thread [{}]: {}", 
                thread.getName(), ex.getMessage(), ex));
        return t;
    }
}

// 线程池创建
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ProductionThreadFactory("order-service")
);
```

## 关联知识点