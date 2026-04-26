---
title: 锁粗化
tags:
  - Java/并发
  - 问答
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# 锁粗化（JIT将多个相邻加锁合并成一个）

## Q1：什么是锁粗化？JIT 为什么要做这个优化？

**A**：锁粗化是 JIT 编译器将**连续多次对同一对象的加锁/解锁操作合并为一次**的优化。

原因：每次加锁/解锁都有一定开销（CAS 操作、内存屏障等），如果在循环中频繁加锁解锁，总开销可能超过临界区本身的开销。合并后只需一次加锁/解锁，显著减少同步开销。

例如在循环中对同一个 list 做 `synchronized(list){ list.add() }`，JIT 会将其粗化为在循环外加一次锁。

---

## Q2：锁粗化和锁细化（减小锁粒度）矛盾吗？

**A**：不矛盾，两者适用不同场景：

- **锁粗化**：适合多个**连续的短临界区**对同一对象加锁，减少加锁/解锁频率
- **锁细化**：适合一个**大的临界区**中只有小部分需要同步，减小锁粒度提高并发度

选择标准：如果临界区很短且连续，应该粗化；如果临界区很长且只有部分需要同步，应该细化。JIT 会自动做出合理判断。

---

## Q3：锁粗化在什么情况下不会发生？

**A**：以下情况 JIT 不会进行锁粗化：

1. 加锁对象不同（对不同对象加锁不会合并）
2. 两次加锁之间有共享变量的读写（其他线程可能需要访问）
3. 代码还未被 JIT 编译（解释执行阶段没有优化）

锁粗化只在 JIT 编译热点代码时进行，且依赖逃逸分析和线程安全分析。

---

## Q4：JIT 锁粗化的阈值是多少？可以手动控制吗？

**A**：

JIT 锁粗化有一个内部阈值（默认约 **5000 次**），当同一对象的 synchronized 块执行次数超过阈值时触发。

```bash
# 禁用锁粗化（测试用）
-XX:-DoConservativeStackFrameHeight

# 锁粗化阈值（JDK 11+ 可调整）
-XX:LoopUnrollMinLimit=1000
```

**注意**：JIT 参数在不同 JDK 版本差异很大，一般不建议手动调整。锁粗化是 JIT 自动行为，开发者只需了解原理即可。

**实战建议**：

```java
// ❌ 低效：循环内加锁
for (Data data : batch) {
    synchronized (this) {
        cache.put(data.getKey(), data.getValue());
    }
}

// ✅ 高效：循环外加锁（手动粗化）
synchronized (this) {
    for (Data data : batch) {
        cache.put(data.getKey(), data.getValue());
    }
}
```

> **提示**：即使 JIT 会做锁粗化，手动粗化代码更清晰，也避免了 JIT 编译前的"解释执行阶段"开销。

---

## Q5：锁粗化 vs 锁细化 vs 分段锁 的区别和应用场景？

**A**：

| 优化策略 | 原理 | 适用场景 | 示例 |
|----------|------|----------|------|
| **锁粗化** | 合并多个相邻细锁 | 多个短临界区连续执行 | 循环中多次加锁 |
| **锁细化** | 缩小锁粒度 | 大临界区中只有部分需要同步 | ConcurrentHashMap |
| **分段锁** | 多个锁分管不同数据段 | 数据可分段，互不干扰 | ConcurrentHashMap JDK7 |

```java
// 锁细化示例：只锁必要部分
public void process(Order order) {
    // 非临界区：无锁
    validate(order);

    // 临界区1：只锁计数器
    synchronized (counterLock) {
        orderCount++;
    }

    // 临界区2：只锁缓存
    synchronized (cacheLock) {
        cache.put(order.getId(), order);
    }

    // 非临界区：无锁
    notify(order);
}

// 分段锁示例（ConcurrentHashMap JDK7）
Segment<K,V>[] segments;  // 16个分段
private int hash(K key) {
    return key.hashCode() % segments.length;  // 不同key落在不同段
}
```

**JDK 8+ 的变化**：ConcurrentHashMap 移除了分段锁，改用 **CAS + synchronized**，锁粒度细化到每个桶的头节点。

---

## 关联知识点

