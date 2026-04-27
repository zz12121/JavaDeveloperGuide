---
tags: [Java并发, 缓存设计, ConcurrentHashMap, 缓存击穿, 热点Key]
module: 17_实际应用与场景
chapter: 01_并发缓存设计
---

# 并发缓存设计

## Q1：如何用 ConcurrentHashMap 实现一个线程安全的缓存？

**A**：

使用 `computeIfAbsent` 实现原子计算：

```java
private final ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();

public Object get(String key) {
    return cache.computeIfAbsent(key, k -> loadFromDB(k));
}
```

`computeIfAbsent` 保证同一个 key 的计算逻辑只执行一次，避免并发场景下重复加载，天然防止缓存击穿。

---

## Q2：什么是缓存穿透、缓存击穿、缓存雪崩？如何解决？

**A**：

| 问题 | 定义 | 解决方案 |
|------|------|---------| 
| **穿透** | 查询不存在的数据，每次打到 DB | 布隆过滤器 / 缓存空值（null） |
| **击穿** | 热点 key 过期，大量并发请求同时打到 DB | `computeIfAbsent` / 互斥锁重建 / 不过期 |
| **雪崩** | 大量 key 同时过期，所有请求打到 DB | 随机过期时间 / 多级缓存 / 永不过期+异步更新 |

**缓存空值防穿透**：
```java
// 查询不存在的 key 时，也缓存一个 null 标记值，防止反复打 DB
cache.computeIfAbsent(key, k -> {
    Object val = loadFromDB(k);
    return val != null ? val : NULL_PLACEHOLDER;  // 用特殊对象替代 null
});
```

---

## Q3：computeIfAbsent 在高并发下有什么注意点？

**A**：

1. **计算函数不要耗时过长**：`computeIfAbsent` 会锁住整个 hash 槽，其他线程等待
2. **不要在计算函数中修改同一个 Map**：可能导致死锁
3. **null 值问题**：`ConcurrentHashMap` 不允许 null 值，计算函数返回 null 不会放入缓存，且会被反复调用
4. **JDK8 vs JDK9+**：JDK9 之前 `computeIfAbsent` 的映射函数是同步的，JDK9+ 优化为无锁读

如果加载非常耗时，建议用 `Future` 或 `CompletableFuture` 替代：
```java
cache.computeIfAbsent(key, k -> CompletableFuture.supplyAsync(() -> loadFromDB(k))).join();
```

---

## Q4：如何处理热点 Key 问题？

**A**：

热点 Key 本质是**单点压力过大**，有两类解决思路：

**方案1：二级缓存（本地 + Redis）**

```java
// 本地 Caffeine 挡大部分流量，Redis 作为共享层
public V get(K key) {
    V value = localCache.getIfPresent(key);  // 1. 先查本地（纳秒级）
    if (value != null) return value;

    value = redis.opsForValue().get(key);    // 2. 再查 Redis（毫秒级）
    if (value != null) {
        localCache.put(key, value);          // 回填本地缓存
        return value;
    }

    return loadFromDB(key);                  // 3. 最后查 DB
}
```

本地缓存设置较短过期（如 5 秒），Redis 设置较长过期（如 30 分钟）。

**方案2：Key 分片**

```java
// 将热点 key 拆成 N 个分片，分散到不同 Redis 节点/槽位
int shard = ThreadLocalRandom.current().nextInt(10);
String shardKey = "hot_key:shard:" + shard;  // 10个分片均摊流量

// 写时：同时写 N 个分片
// 读时：随机选一个分片读取
```

**方案3：防止本地缓存穿透（互斥锁+双重检查）**

```java
// 热点 key 过期瞬间，只允许一个线程重建缓存
ReentrantLock lock = locks.computeIfAbsent(key, k -> new ReentrantLock());
lock.lock();
try {
    V value = cache.get(key);  // 双重检查
    if (value == null) {
        value = loadFromSource(key);
        cache.put(key, value);
    }
    return value;
} finally {
    lock.unlock();
}
```

---

## Q5：CAS 方式更新缓存值有什么用？

**A**：

`ConcurrentHashMap.replace(key, oldValue, newValue)` 是 CAS 语义的原子更新：

```java
// 场景：多线程并发更新配置缓存，防止旧值覆盖新值
ConcurrentHashMap<String, Config> configCache = new ConcurrentHashMap<>();

public boolean updateConfig(String key, Config expected, Config newConfig) {
    // 只有当前值等于 expected 时才更新，否则返回 false
    return configCache.replace(key, expected, newConfig);
}

// 乐观锁重试模式
public void safeUpdate(String key, Function<Config, Config> updater) {
    Config current;
    do {
        current = configCache.get(key);
    } while (!configCache.replace(key, current, updater.apply(current)));
}
```

**用途**：
- 防止 ABA 问题（旧值校验）
- 配置热更新（版本比对）
- 多线程下的无锁缓存更新（避免 synchronized 全局锁）

---

## Q6：本地缓存和 Redis 的一致性如何保证？

**A**：

**问题**：更新数据库后，本地缓存和 Redis 可能出现不一致。

**方案1：Canal + MQ 订阅 binlog**（最终一致，推荐）

```
DB 更新 → binlog → Canal 解析 → MQ → 消费者删除/更新 Redis 和本地缓存
```

**方案2：先更新 DB，再删除 Redis（Cache-Aside）**

```java
// 写操作：先写 DB，再删缓存（不是更新缓存！）
@Transactional
public void updateUser(User user) {
    userDao.update(user);          // 1. 更新数据库
    redis.delete("user:" + id);    // 2. 删除 Redis 缓存（下次读时重建）
    // 不清本地缓存的话，本地缓存等待自然过期（TTL 5s）
}
```

**方案3：TTL 兜底**（最简单，允许短暂不一致）

- 本地缓存设置短 TTL（5~10 秒）
- Redis 设置中等 TTL（5~30 分钟）
- 数据修改后自然失效，适合容忍短暂不一致的场景
