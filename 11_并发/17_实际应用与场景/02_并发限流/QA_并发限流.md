---
tags: [Java并发, 限流, RateLimiter, 令牌桶, 漏桶, Semaphore]
module: 17_实际应用与场景
chapter: 02_并发限流
---

# 并发限流

## Q1：有哪些常见的限流方案？

**A**：

| 方案 | 原理 | 适用场景 |
|------|------|---------| 
| **Semaphore** | 控制最大并发数 | 限制同时执行的请求数 |
| **RateLimiter（令牌桶）** | 固定速率生成令牌，允许突发 | 限制 QPS，允许瞬时高峰 |
| **漏桶** | 固定速率处理请求，强制匀速 | 流量整形，严格匀速输出 |
| **滑动窗口** | 固定时间窗口内计数 | Redis 分布式限流 |
| **Sentinel** | 综合限流框架 | 微服务链路级限流 |

---

## Q2：Semaphore 和 RateLimiter 有什么区别？

**A**：

- **Semaphore**：限制**同时执行**的线程数（并发数），不关心速率
- **RateLimiter**：限制**单位时间内**的请求数（QPS），关心速率

```java
// Semaphore：最多 10 个线程同时在处理中
Semaphore semaphore = new Semaphore(10);
if (semaphore.tryAcquire()) {
    try { handleRequest(); }
    finally { semaphore.release(); }
}

// RateLimiter：每秒最多 100 个请求
RateLimiter limiter = RateLimiter.create(100);
if (limiter.tryAcquire(500, TimeUnit.MILLISECONDS)) {
    handleRequest();
}
```

**场景举例**：DB 连接池限制并发用 Semaphore；API 防刷用 RateLimiter。

---

## Q3：令牌桶和漏桶算法有什么本质区别？

**A**：

**核心差异**：

| 维度 | 令牌桶 | 漏桶 |
|------|--------|------|
| **突发流量** | ✅ 允许（积累令牌） | ❌ 强制匀速，不允许突发 |
| **限制维度** | 平均速率（允许短时超过） | 瞬时速率（严格均匀输出） |
| **适用场景** | API 限流、秒杀 | 短信/邮件发送、资金转账 |
| **JDK 支持** | Guava `RateLimiter` | 无原生，手动实现 |

```
令牌桶：
  生成令牌 → [桶(容量10)] → 请求消耗令牌
  积累了10个令牌 → 可以瞬间处理10个请求（允许突发！）

漏桶：
  请求进入 → [桶(容量N)] → 固定速率流出
  无论多少请求进来，始终每秒输出10个（强制匀速！）
```

**选择原则**：
- 允许流量突发（如秒杀） → 令牌桶
- 必须严格均匀（如短信发送，防止瞬间大量下发） → 漏桶

---

## Q4：分布式限流怎么实现？

**A**：

1. **Redis + Lua（滑动窗口）**：
```java
// Lua 原子操作：清理过期记录 + 计数 + 判断
String luaScript = """
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local limit = tonumber(ARGV[3])
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    local count = redis.call('ZCARD', key)
    if count < limit then
        redis.call('ZADD', key, now, now)
        redis.call('EXPIRE', key, window/1000)
        return 1
    end
    return 0
    """;
```

2. **Redis + 令牌桶（Lua）**：生成速率和消耗操作原子化

3. **Sentinel（阿里）**：开箱即用，支持 QPS/并发数/热点参数/链路等多种规则

4. **Spring Cloud Gateway**：内置 `RequestRateLimiter` 过滤器，基于 Redis 滑动窗口

**单机** → Semaphore/RateLimiter；**分布式** → Redis/Sentinel。

---

## Q5：RateLimiter 的预热（warmup）是什么？什么时候用？

**A**：

`RateLimiter.create(rate, warmupPeriod, unit)` 创建预热型限速器：

```java
// 预热30秒：刚启动时速率低，30秒内逐步提升到 100 QPS
RateLimiter warmupLimiter = RateLimiter.create(100, 30, TimeUnit.SECONDS);
```

**原理**：预热期内，令牌生成速率从 `rate/3`（或更低）线性增长到 `rate`，而非一开始就满速。

**适用场景**：
- 应用冷启动（缓存未预热，数据库连接池未满）
- 防止突然大流量打垮刚启动的服务
- 需要逐步预热连接池/缓存的场景

**Guava RateLimiter 的 acquire vs tryAcquire**：
```java
limiter.acquire();                        // 阻塞等待，直到获取到令牌
limiter.tryAcquire();                     // 立即返回，无令牌时返回 false
limiter.tryAcquire(500, TimeUnit.MILLISECONDS);  // 最多等500ms
```

---

## Q6：生产环境中限流的完整实现方案？

**A**：

**多层限流架构**：

```
Nginx（连接数限流）→ Gateway（QPS限流）→ 微服务（业务维度限流）
```

```java
// 微服务层：注解 + AOP 方式
@RateLimiter(qps = 100, timeout = 500)  // 自定义注解
public Result queryProduct(String skuId) { ... }

// AOP 实现
@Around("@annotation(rateLimiter)")
public Object around(ProceedingJoinPoint pjp, RateLimiter rateLimiter) throws Throwable {
    String key = buildKey(pjp);  // 按接口/用户维度
    if (!redisLimiter.tryAcquire(key, rateLimiter.qps(), rateLimiter.timeout())) {
        throw new RateLimitException("请求过于频繁，请稍后重试");
    }
    return pjp.proceed();
}
```

**限流降级策略**：
```java
// 超过限流时，选择合适的降级方案
if (!limiter.tryAcquire()) {
    // 1. 快速失败：直接返回错误
    return Result.fail("系统繁忙");
    
    // 2. 读缓存：降级为读旧数据
    return cacheService.getStaleData(key);
    
    // 3. 排队等待：适合可容忍延迟的场景
    limiter.acquire();  // 阻塞等待
}
```
