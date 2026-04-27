
# ThreadLocal vs 锁

## 核心结论

`ThreadLocal` 是**以空间换时间**——每个线程持有一份数据副本，无需同步。锁是**以时间换空间**——多线程共享数据，通过同步控制访问。两者解决线程安全问题的思路完全不同。

## 深度解析

### 核心区别

| 维度 | ThreadLocal | 锁（synchronized/Lock） |
|------|------------|----------------------|
| **核心思想** | 空间换时间（数据隔离） | 时间换空间（互斥访问） |
| **数据存储** | 每线程独立副本 | 共享同一份数据 |
| **性能** | 无竞争，读写都很快 | 竞争时线程阻塞等待 |
| **内存开销** | 每线程一个副本，内存大 | 只存一份数据，内存小 |
| **适用场景** | 线程间无共享需求 | 线程间必须共享数据 |
| **一致性** | 各线程数据独立，无一致性问题 | 保证共享数据一致性 |
| **死锁风险** | 无 | 有 |

### 原理对比

```
ThreadLocal（空间换时间）：
线程1 → ThreadLocalMap → value_1
线程2 → ThreadLocalMap → value_2
线程3 → ThreadLocalMap → value_3
（各自独立，无需同步）

锁（时间换空间）：
线程1 ──┐
线程2 ──┼──→ 共享数据 ← 锁控制同时只有一个线程访问
线程3 ──┘
（共享数据，需要同步）
```

### 代码对比

```java
// 方式一：ThreadLocal（无锁）
private static final ThreadLocal<SimpleDateFormat> sdf = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
// 每个线程用自己的实例，无竞争

// 方式二：synchronized（加锁）
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
public synchronized String format(Date date) {
    return sdf.format(date);
}
// 所有线程排队访问同一个实例
```

### 选择指南

| 场景 | 推荐方案 |
|------|---------|
| 对象创建开销小、无共享需求 | ThreadLocal |
| 对象创建开销大（如数据库连接） | 锁 + 共享 |
| 需要线程间共享状态 | 锁 |
| 需要保证一致性（如计数器） | 锁 / 原子类 |
| 上下文传递（User/TraceId） | ThreadLocal |
| 读多写少 | ReentrantReadWriteLock |

### 局限性

- **ThreadLocal 不能解决共享问题**：如果多个线程需要操作同一份数据，ThreadLocal 无法胜任
- **锁有性能瓶颈**：高并发下锁竞争严重，吞吐量下降
- **最佳实践**：能不共享就不共享（ThreadLocal），必须共享才用锁

## 关联知识点