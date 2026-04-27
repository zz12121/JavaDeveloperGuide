
# ThreadLocal vs 锁

## Q1：ThreadLocal 和锁有什么区别？

**A**：

| 维度 | ThreadLocal | 锁 |
|------|------------|---|
| 核心思想 | 空间换时间，数据隔离 | 时间换空间，互斥访问 |
| 数据存储 | 每线程独立副本 | 共享同一份数据 |
| 性能 | 无竞争，读写极快 | 竞争时阻塞等待 |
| 内存 | 每线程一个副本，内存占用大 | 一份数据，内存占用小 |
| 适用 | 线程间无需共享 | 线程间必须共享 |

一句话总结：**ThreadLocal 让你不共享，锁让你安全地共享**。

---

## Q2：什么时候用 ThreadLocal，什么时候用锁？

**A**：

- **用 ThreadLocal**：数据不需要在线程间共享，每个线程用自己的副本即可（如 SimpleDateFormat、用户上下文、TraceId）
- **用锁**：多个线程必须操作同一份数据，需要保证一致性（如计数器、余额更新、库存扣减）

能不共享就不共享，必须共享才用锁。优先考虑 ThreadLocal，解决不了再用锁。

---

## Q3：ThreadLocal 能完全替代锁吗？

**A**：

不能。ThreadLocal 只解决**数据隔离**问题，不解决**共享数据的并发安全**问题。

例如：多线程对一个共享计数器 `count++`，用 ThreadLocal 每个线程各有各的 count，无法得到全局总数。这种场景必须用锁或原子类。

---

## Q4：ThreadLocal 和 synchronized 能一起用吗？

**A**：

可以，它们解决不同维度的问题。例如 Spring 的 `TransactionSynchronizationManager` 内部用 `ThreadLocal` 存储事务资源，同时数据库连接本身又是线程安全的。

但一般没必要同时使用——如果数据已经用 ThreadLocal 隔离了，就不需要再加锁。

---
```java
// ThreadLocal：空间换时间，数据隔离
ThreadLocal<SimpleDateFormat> sdfHolder = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd")
);
// 每个线程有自己的 SimpleDateFormat 实例，无需加锁

// 锁：时间换空间，互斥访问
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
synchronized (sdf) {
    sdf.parse("2024-01-01");  // 加锁保证线程安全
}

// 选择标准：
// 能不共享就不共享 → ThreadLocal
// 必须共享才加锁 → synchronized / Lock
```

