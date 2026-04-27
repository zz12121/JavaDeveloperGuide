
# TransmittableThreadLocal

## 核心结论

`TransmittableThreadLocal`（TTL）是阿里开源的框架，解决 `InheritableThreadLocal` 在线程池中**值无法正确传递**的问题。核心思路是**在提交任务时捕获值，执行任务时恢复值**。

## 深度解析

### 工作原理

```
主线程                    TTL                      线程池线程
──────                    ────                      ─────────
set("user-B")             
                          capture() → 快照 {"user-B": "user-B"}
                          wrap(runnable) → TtlRunnable
submit(TtlRunnable)       
                                                      
                                                     run() 前：replay() → 设置 {"user-B": "user-B"}
                                                     run() 执行 → get() = "user-B" ✅
                                                     run() 后：restore() → 恢复原值
```

### 使用方式

```java
// 1. 替换 InheritableThreadLocal → TransmittableThreadLocal
private static final TransmittableThreadLocal<String> CONTEXT = 
    new TransmittableThreadLocal<>();

// 2. 使用 TtlExecutors 包装线程池
ExecutorService pool = TtlExecutors.getTtlExecutorService(
    Executors.newFixedThreadPool(1)
);

// 3. 正常使用
CONTEXT.set("user-A");
pool.submit(() -> {
    System.out.println("任务1: " + CONTEXT.get()); // user-A ✅
});

CONTEXT.set("user-B");
pool.submit(() -> {
    System.out.println("任务2: " + CONTEXT.get()); // user-B ✅
});
```

### 核心机制：捕获与恢复

| 阶段 | 方法 | 作用 |
|------|------|------|
| 任务提交 | `capture()` | 快照当前线程所有 TTL 值 |
| 任务包装 | `wrap(runnable)` | 将快照绑定到 Runnable |
| 任务执行前 | `replay(captured)` | 用快照覆盖子线程 TTL 值 |
| 任务执行后 | `restore(backup)` | 恢复子线程原来的 TTL 值 |

### TTL vs InheritableThreadLocal

| 特性 | InheritableThreadLocal | TransmittableThreadLocal |
|------|----------------------|------------------------|
| 值传递时机 | 创建线程时 | 每次任务提交时 |
| 线程池兼容 | ❌ 不兼容 | ✅ 完美兼容 |
| 值恢复 | 不支持 | 执行完恢复原值 |
| 额外依赖 | JDK 自带 | 需引入 TTL 库 |
| 性能影响 | 几乎无 | 有少量开销（快照/恢复） |

### Maven 依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.5</version>
</dependency>
```

### Java Agent 方式（无侵入）

```bash
# 启动时加 Agent，自动包装所有线程池，无需改代码
java -javaagent:transmittable-thread-local-2.14.5.jar -jar app.jar
```

## 关联知识点

