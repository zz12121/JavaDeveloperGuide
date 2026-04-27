---
tags: [Java并发, 连接池, HikariCP, Druid, 连接泄漏]
module: 17_实际应用与场景
chapter: 07_连接池管理
---

# 连接池管理

## Q1：连接池的原理是什么？

**A**：

连接池预先创建一组数据库连接放入池中，应用需要时从池中获取（borrow），用完后归还（return），而不是每次新建和销毁。

核心组件：
- **BlockingQueue**：存储空闲连接
- **Semaphore**：控制最大连接数（获取时 acquire，归还时 release）
- **连接验证**：借出/归还时检测连接有效性

好处：减少连接创建/销毁开销，控制最大并发数防止 DB 过载。

---

## Q2：HikariCP 为什么是最快的连接池？

**A**：

HikariCP 有三大核心优化：

**1. ConcurrentBag（无锁并发容器）**：
- ThreadLocal 优先：每个线程先尝试从自己的"私有"列表获取连接，无竞争
- 私有列表没有时，再从共享队列抢（CAS 更新状态）

**2. FastList（替代 ArrayList）**：
- 自定义 List，`remove` 操作从尾部倒序扫描（Connection 通常后借先还）
- 减少数组元素移动开销

**3. 字节码代理（Javassist）**：
- 编译期生成 `Connection` 代理类，无需运行时反射
- 比 JDK 动态代理快约 28 倍

```
测试数据（同等条件）：
HikariCP:  ~60,000 ops/ms
Druid:     ~20,000 ops/ms  
C3P0:       ~3,000 ops/ms
```

---

## Q3：HikariCP 和 Druid 怎么选？

**A**：

| 维度 | HikariCP | Druid |
|------|---------|-------|
| 性能 | ⭐⭐⭐ 最佳 | ⭐⭐ 略低 |
| SQL 监控 | ❌ 无 | ✅ SQL 统计、慢查询 |
| 连接泄漏检测 | ✅ 日志告警 | ✅ 自动回收 |
| 防 SQL 注入 | ❌ | ✅ WallFilter |
| Spring Boot 默认 | ✅ | 需手动引入 |
| 社区活跃度 | 高（brettwooldridge） | 高（阿里） |

**选择建议**：
- 追求极致性能，Spring Boot 项目 → **HikariCP**
- 需要 SQL 监控、慢查询统计、运维可视化 → **Druid**

---

## Q4：连接池大小如何设置？

**A**：

**HikariCP 官方公式**（PostgreSQL 团队验证）：
```
pool_size = CPU核数 × 2 + 有效磁盘数
```

**实际经验**：
```java
// IO 密集型（大量 DB 查询）：CPU核数 × 2 左右
config.setMaximumPoolSize(Runtime.getRuntime().availableProcessors() * 2);

// 通用经验值：10~20 个连接对大多数 OLTP 场景足够
config.setMaximumPoolSize(20);
```

**连接过多的问题**：
- DB 服务器连接过多 → 大量上下文切换 → 性能反而下降
- 数据库处理是串行的，100 个连接等 SQL 执行，不如 20 个连接快速轮转

**调优步骤**：
1. 监控连接池 `activeConnections` 和 `pendingThreads`
2. 如果 `pendingThreads > 0`，说明连接不够，适当增加
3. 如果 `activeConnections` 远低于 `maximumPoolSize`，说明连接过多，可以减少

---

## Q5：连接泄漏怎么预防和检测？

**A**：

**预防**：
```java
// 最佳实践：try-with-resources，确保连接一定归还
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.executeQuery();
} // 自动 close，归还连接池
```

**HikariCP 泄漏检测**：
```yaml
# application.yml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 60000  # 60秒未归还打印告警日志+堆栈
```

打印的堆栈会显示是哪行代码借走了连接，帮助快速定位泄漏位置。

**Druid 自动回收**：
```java
config.setRemoveAbandoned(true);
config.setRemoveAbandonedTimeout(30); // 30秒自动回收超时连接
```

**常见泄漏场景**：
1. 异常路径未关闭 Connection（没有 try-with-resources）
2. Spring 事务中手动获取连接但未归还
3. 测试代码忘记关闭连接

---

## Q6：连接池的 keepAlive 和 maxLifetime 有什么用？

**A**：

**问题背景**：DB 服务器（如 MySQL）有 `wait_timeout`（默认 8 小时），空闲连接超过这个时间会被服务端强制断开。应用再次使用时报 "Connection closed" 或 "Broken pipe"。

**解决方案**：

```java
// maxLifetime：连接的最大存活时间，到期后连接池主动销毁并重建
// 必须 < DB wait_timeout，建议 DB 超时的 75%
config.setMaxLifetime(1800000);  // 30分钟（DB wait_timeout = 8小时时，取 30分钟即可）

// keepaliveTime（HikariCP 4.x+）：定期发送心跳 ping（比 testOnBorrow 开销小）
// 空闲连接每隔 keepaliveTime 发一次 ping，保持活跃
config.setKeepaliveTime(120000);  // 2分钟

// connectionTestQuery（旧版本替代 keepalive）
config.setConnectionTestQuery("SELECT 1");  // 借出连接时验证
```

**优先级建议**：优先用 `keepaliveTime`（后台定期，不影响借出速度），不要用 `testOnBorrow`（每次借出都发 SQL，额外延迟）。
