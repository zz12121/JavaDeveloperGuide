---
tags: [Java并发, 单例模式, DCL, 高并发初始化, CountDownLatch]
module: 17_实际应用与场景
chapter: 06_高并发初始化
---

# 高并发初始化

## Q1：如何并行加载多个资源并等待全部完成后启动服务？

**A**：

使用 `CompletableFuture.allOf`（推荐）：

```java
CompletableFuture<Void> db     = CompletableFuture.runAsync(this::loadDatabase, executor);
CompletableFuture<Void> cache  = CompletableFuture.runAsync(this::loadCache, executor);
CompletableFuture<Void> config = CompletableFuture.runAsync(this::loadConfig, executor);

CompletableFuture.allOf(db, cache, config)
    .orTimeout(30, TimeUnit.SECONDS)
    .whenComplete((v, ex) -> {
        if (ex == null) startServer();
        else { log.error("初始化失败", ex); System.exit(1); }
    });
```

比 `CountDownLatch` 更灵活：支持超时、异常回调、链式组合。`CountDownLatch` 适合不需要结果处理的简单场景。

---

## Q2：CountDownLatch 和 CyclicBarrier 在初始化场景怎么选？

**A**：

- **CountDownLatch**：一次性的"等待 N 个任务完成"，**不可重置**，适合应用启动初始化
- **CyclicBarrier**：可重复使用的"多线程到齐后一起执行"，**可复用**，适合多阶段初始化

```java
// CountDownLatch：启动时等资源加载
CountDownLatch latch = new CountDownLatch(3);
executor.submit(() -> { loadDB(); latch.countDown(); });
latch.await(30, TimeUnit.SECONDS);
startServer();  // 全部加载完才执行

// CyclicBarrier：多阶段（基础资源 → 业务模块）
CyclicBarrier barrier = new CyclicBarrier(3, () -> log.info("阶段完成"));
// 每个线程到达 barrier.await() 后才一起进入下一阶段
```

选择依据：只需等一次 → `CountDownLatch`；需要多阶段同步 → `CyclicBarrier`。

---

## Q3：DCL 双重检查锁为什么必须用 volatile？

**A**：

`instance = new Singleton()` 不是原子操作，底层分 3 步：
1. 分配内存空间
2. 初始化对象（执行构造函数）
3. 将引用 `instance` 指向内存地址

没有 `volatile` 时，JIT/CPU 可能重排序为 **1→3→2**：

```
线程A：1→3（instance已非null）→ 正在执行步骤2初始化...
线程B：第一重检查 instance != null → 直接return → 拿到未初始化的对象 → NPE！
```

`volatile` 写操作有 StoreStore + StoreLoad 屏障，**强制 1→2→3 顺序**，且写完后其他线程立即可见。

---

## Q4：静态内部类（Holder模式）单例为什么是线程安全的？

**A**：

```java
public class Singleton {
    private Singleton() {}

    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

**线程安全原因**：

JVM 规范要求：类的初始化（`<clinit>` 方法）由 JVM 加锁保证只执行一次，多个线程竞争时其余线程阻塞等待。

- `Holder` 类只在 `getInstance()` 第一次调用时才被 ClassLoader 加载
- 加载过程 JVM 自动加锁，`INSTANCE` 只会被初始化一次
- 之后的调用直接读取已初始化完成的 `INSTANCE`，无锁开销

**优势**：实现最简洁，兼具懒加载 + 线程安全，无需 `synchronized` 或 `volatile`。

---

## Q5：枚举单例为什么是"最安全"的？

**A**：

```java
public enum Singleton {
    INSTANCE;
    public void doWork() { /* ... */ }
}
```

枚举单例有三重保障，普通单例做不到：

| 安全维度 | 普通单例（DCL/Holder） | 枚举单例 |
|---------|---------------------|---------|
| 线程安全 | ✅（需额外手段） | ✅（JVM类加载） |
| 防反射攻击 | ❌ 反射可调 private 构造器 | ✅ 枚举构造器调用抛 `IllegalArgumentException` |
| 防序列化攻击 | ❌ 反序列化默认创建新对象 | ✅ JVM保证枚举序列化复用同一实例 |

**反射攻击示例（对普通单例有效，对枚举无效）**：
```java
// 普通单例可被反射攻破
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton another = c.newInstance(); // 创建了第二个实例！

// 枚举会抛：Cannot reflectively create enum objects
```

**适用场景**：不需要懒加载、对安全性要求高的场景（如工具类单例、Spring Bean 替代方案）。

---

## Q6：初始化时某个模块加载失败，如何安全处理？

**A**：

**方案一：CompletableFuture + exceptionally**（推荐）

```java
CompletableFuture<Void> db = CompletableFuture.runAsync(() -> {
    loadDatabase();  // 抛异常会传播到 allOf
}, executor);

CompletableFuture.allOf(db, cache, config)
    .exceptionally(ex -> {
        log.error("初始化失败: {}", ex.getMessage());
        System.exit(1);  // 关键模块失败直接退出
        return null;
    });
```

**方案二：CountDownLatch + AtomicBoolean 标记失败**

```java
CountDownLatch latch = new CountDownLatch(3);
AtomicBoolean failed = new AtomicBoolean(false);

executor.submit(() -> {
    try { loadDatabase(); }
    catch (Exception e) { failed.set(true); log.error("DB初始化失败", e); }
    finally { latch.countDown(); }
});

latch.await(30, TimeUnit.SECONDS);
if (failed.get()) throw new RuntimeException("初始化失败，服务终止");
```

推荐使用 `CompletableFuture`，异常传播更自然，代码更简洁。

---

## Q7：Spring Boot 应用启动时的初始化最佳实践？

**A**：

```java
@Component
public class ApplicationStartup implements ApplicationRunner {

    @Autowired
    private ExecutorService startupExecutor;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 并行预热：本地缓存 + 路由规则 + 预置数据
        CompletableFuture<Void> warmCache =
            CompletableFuture.runAsync(this::warmUpCache, startupExecutor);
        CompletableFuture<Void> loadRoute =
            CompletableFuture.runAsync(this::loadRouteRules, startupExecutor);

        try {
            CompletableFuture.allOf(warmCache, loadRoute)
                .get(60, TimeUnit.SECONDS);
            log.info("应用预热完成，开始对外提供服务");
        } catch (TimeoutException e) {
            log.warn("预热超时，降级启动");
        } catch (ExecutionException e) {
            log.error("预热异常", e.getCause());
            // 非关键资源可选择降级，关键资源应终止启动
        }
    }
}
```

**关键原则**：
1. 使用独立线程池（`startupExecutor`），避免阻塞 Spring 主线程
2. 设置合理超时，防止启动挂起
3. 区分关键/非关键资源：关键资源失败 → `System.exit(1)`；非关键 → 降级继续
