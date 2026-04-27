
# 虚拟线程 vs 平台线程

## 核心结论

虚拟线程是 JDK 对传统平台线程的升级，保持 API 兼容的同时极大提升了并发能力。核心区别在于**调度方式和资源占用**：平台线程由 OS 调度（重量级），虚拟线程由 JVM 调度（轻量级）。

## 深度解析

### 全面对比

| 维度 | 平台线程 | 虚拟线程 |
|------|---------|---------|
| **调度方** | 操作系统 | JVM（ForkJoinPool） |
| **栈大小** | 默认 1MB（可调 `-Xss`） | **最大 1MB**，初始仅 ~512B（按需增长） |
| **创建成本** | 高（系统调用） | 极低（JVM 内部分配） |
| **数量上限** | 几千（受 OS 限制） | 百万级 |
| **阻塞行为** | 阻塞 OS 线程 | 挂起虚拟线程，释放载体线程 |
| **上下文切换** | 慢（内核态切换） | 快（用户态切换） |
| **ThreadLocal** | 正常支持 | 支持，但百万线程时内存压力大 |
| **synchronized** | 正常支持 | JDK 21+ 已优化，阻塞时挂起 |
| **适用场景** | CPU 密集型 | IO 密集型 |

### 调度模型对比

```
平台线程（1:1 模型）：
Thread-1 → OS Thread-1
Thread-2 → OS Thread-2
Thread-3 → OS Thread-3
（每个 Java 线程对应一个 OS 线程）

虚拟线程（M:N 模型）：
VT-1 ─┐
VT-2 ─┼─→ Carrier Thread (ForkJoinPool)
VT-3 ─┤
VT-4 ─┘
（大量虚拟线程复用少量载体线程）
  ↓
  载体线程数 = CPU 核数（如 8 核 = 8 个载体线程）
  虚拟线程数 = 无限制（如 100 万）
```

### 阻塞行为差异

```java
// 平台线程
Thread thread = new Thread(() -> {
    Thread.sleep(5000); // OS 线程被阻塞，浪费资源
});

// 虚拟线程
Thread.startVirtualThread(() -> {
    Thread.sleep(5000); // 虚拟线程挂起，载体线程可服务其他虚拟线程
});
```

### 迁移成本

```java
// 旧代码无需修改，只需改创建方式
// 旧：
new Thread(task).start();
executor.submit(task);

// 新：
Thread.startVirtualThread(task);
Executors.newVirtualThreadPerTaskExecutor().submit(task);
```

