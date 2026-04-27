---
tags: [Java并发, 异步日志, Log4j2, Disruptor, MDC, 链路追踪]
module: 17_实际应用与场景
chapter: 08_并发日志
---

# 并发日志

## Q1：高并发下日志系统如何保证性能？

**A**：

1. **异步日志**：Log4j2 AsyncLogger 基于 Disruptor 无锁环形队列，日志写入由独立线程完成，业务线程不阻塞
2. **批量写入**：积累一定量日志后批量刷盘，减少 IO 次数
3. **无锁队列**：Disruptor 的 Ring Buffer 使用 CAS + 序号机制，避免锁竞争
4. **合理日志级别**：生产环境使用 INFO/WARN/ERROR，避免 DEBUG 大量输出
5. **避免字符串拼接**：用占位符 `log.info("user:{}", id)` 替代 `log.info("user:" + id)`

---

## Q2：AsyncAppender 和 AsyncLogger 有什么区别？

**A**：

| 维度 | AsyncAppender（Logback） | AsyncLogger（Log4j2） |
|------|------------------------|---------------------|
| 底层队列 | `BlockingQueue`（有锁） | **Disruptor**（无锁） |
| 性能 | 中 | 极高（快 6~10 倍） |
| 配置方式 | 包裹同步 Appender | 直接声明 AsyncLogger/AsyncRoot |
| 队列满策略 | 丢弃低级别日志 | WaitStrategy 阻塞/丢弃 |

```xml
<!-- Logback AsyncAppender：包裹式，改造成本低 -->
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>   <!-- 不丢日志 -->
    <appender-ref ref="FILE"/>
</appender>
```

```xml
<!-- Log4j2 全局异步（推荐，性能最高）-->
<!-- 通过 JVM 参数开启：-Dlog4j2.contextSelector=...AsyncLoggerContextSelector -->
<AsyncRoot level="info">
    <AppenderRef ref="File"/>
</AsyncRoot>
```

**选择建议**：新项目用 Log4j2 AsyncLogger；已有 Logback 项目成本改造低时用 AsyncAppender。

---

## Q3：Disruptor 为什么比 BlockingQueue 快？

**A**：

| 维度 | BlockingQueue | Disruptor |
|------|--------------|-----------| 
| 数据结构 | 链表/数组 | Ring Buffer（环形数组） |
| 并发控制 | Lock / synchronized | CAS + Sequence 序号机制 |
| 内存模型 | 每次创建事件对象 | 预分配，避免 GC |
| 缓存友好 | 差（链表指针跳跃） | 好（数组连续内存） |
| 伪共享 | 未处理 | @Contended 缓存行填充 |

Disruptor 核心优化：**预分配内存避免 GC**、**环形数组缓存行对齐**、**消除伪共享**、**生产者/消费者无锁协调**。

---

## Q4：MDC 在线程池中有什么问题？如何解决？

**A**：

MDC（Mapped Diagnostic Context）底层使用 `ThreadLocal`，线程池复用线程时 MDC 值不会被清理，导致**日志中的 TraceId 串线程**（线程 A 的 TraceId 出现在线程 B 的日志里）。

**解决方案**：

```java
// 方案1：最可靠——在 finally 中 MDC.clear()
public void doFilter(HttpServletRequest req, HttpServletResponse resp, FilterChain chain) {
    MDC.put("traceId", UUID.randomUUID().toString());
    try {
        chain.doFilter(req, resp);
    } finally {
        MDC.clear();  // 必须在 finally 清理！
    }
}

// 方案2：手动传递（提交线程池任务时复制 MDC）
Map<String, String> mdcContext = MDC.getCopyOfContextMap();
executor.submit(() -> {
    MDC.setContextMap(mdcContext);  // 恢复 MDC
    try { doWork(); }
    finally { MDC.clear(); }
});

// 方案3：TransmittableThreadLocal（阿里 TTL）
// 替换 MDC 的底层 ThreadLocal，自动在线程池中传递
// 引入 transmittable-thread-local 依赖后，用 TtlExecutors 包装线程池
ExecutorService ttlExecutor = TtlExecutors.getTtlExecutorService(executor);
```

---

## Q5：日志队列满了会怎样？如何配置？

**A**：

**Logback AsyncAppender 默认行为**：队列剩余容量 < 20% 时，丢弃 TRACE/DEBUG/INFO 级别日志（只保留 WARN/ERROR）。

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <!-- discardingThreshold=0：永不丢弃（可能导致业务线程阻塞） -->
    <!-- discardingThreshold=20：默认，剩余 20% 时开始丢低级别日志 -->
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>true</neverBlock>   <!-- true=队列满时直接丢，不阻塞业务线程 -->
</appender>
```

**Log4j2 AsyncLogger 配置**：
```xml
<!-- log4j2.xml -->
<AsyncRoot level="info" includeLocation="false">  <!-- includeLocation=false，不记录行号（快 5 倍）-->
    <AppenderRef ref="File"/>
</AsyncRoot>
```

**生产建议**：
- 队列大小设为 2 的幂次（1024、4096）
- `includeCallerData/includeLocation=false`（获取调用栈堆帧很慢）
- 监控队列水位（`log4j2.asyncLoggerRingBufferSize` 对应 Disruptor 容量）

---

## Q6：如何实现分布式链路追踪中的日志关联？

**A**：

**核心思路**：在请求入口注入 TraceId，通过 MDC 传递到所有日志语句，日志输出中携带 TraceId，便于跨服务追踪。

```java
// 1. 网关/Filter 入口注入 TraceId
@Component
public class TraceFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        HttpServletRequest request = (HttpServletRequest) req;
        // 从上游服务的请求头取，没有则生成新的
        String traceId = Optional.ofNullable(request.getHeader("X-Trace-Id"))
            .orElse(UUID.randomUUID().toString().replace("-", ""));
        MDC.put("traceId", traceId);
        // 向下游传递（通过 HTTP Header）
        try { chain.doFilter(req, resp); }
        finally { MDC.clear(); }
    }
}
```

```xml
<!-- 2. logback.xml 日志格式中引用 traceId -->
<pattern>%d{HH:mm:ss.SSS} [%thread] [%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
```

```java
// 3. 调用下游 HTTP 服务时传递 TraceId（RestTemplate 拦截器）
restTemplate.getInterceptors().add((req, body, execution) -> {
    req.getHeaders().set("X-Trace-Id", MDC.get("traceId"));
    return execution.execute(req, body);
});
```

**完整链路效果**：
```
2026-04-27 21:30:00 [http-nio-8080-exec-1] [abc123] INFO  UserController - 查询用户: 1001
2026-04-27 21:30:00 [http-nio-8080-exec-1] [abc123] INFO  UserService    - 命中缓存
2026-04-27 21:30:01 [http-nio-8080-exec-2] [def456] INFO  UserController - 查询用户: 1002
```

所有 `[abc123]` 的日志属于同一请求，可以在 ELK/Loki 中过滤 `traceId=abc123` 查看完整链路。
