---
title: CAS在JUC中的应用
tags:
  - Java/并发
  - 问答
  - 场景型
module: 05_CAS
created: 2026-04-18
---

# CAS在JUC中的应用（AtomicInteger/AtomicLong/AtomicReference/AtomicMarkableReference）

## Q1：JUC 中哪些组件用到了 CAS？

**A**：CAS 是 JUC 的基石，几乎所有无锁/轻量级并发组件都依赖它：

| 组件 | CAS 用途 |
|------|---------|
| AtomicInteger/AtomicLong | 自增/自减/更新操作 |
| AtomicReference | 对象引用原子更新 |
| ReentrantLock | CAS 更新 AQS 的 state 字段 |
| ConcurrentHashMap | CAS 初始化 table、CAS 插入空槽 |
| ThreadPoolExecutor | CAS 更新 ctl（控制状态+线程数） |
| CountDownLatch | CAS 递减 count |
| Semaphore | CAS 递减/递增 permits |
| FutureTask | CAS 更新任务状态 |

---

## Q2：AtomicInteger 的 incrementAndGet 底层是怎么实现的？

**A**：通过 Unsafe 的 `getAndAddInt` + 自旋 CAS：

```java
// AtomicInteger.incrementAndGet()
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}

// Unsafe.getAndAddInt（JDK9+）
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // volatile 读
    } while (!compareAndSwapInt(        // CAS 更新
        o, offset, v, v + delta
    ));
    return v;
}
```

1. `getIntVolatile`：volatile 语义读取当前值（保证可见性）
2. `compareAndSwapInt`：CAS 原子更新（保证原子性）
3. 失败则自旋重试

---

## Q3：ConcurrentHashMap 的 put 操作中 CAS 起什么作用？

**A**：JDK8 ConcurrentHashMap 的 put 分三步，前两步用 CAS：

1. **CAS 初始化**：多线程同时 initTable 时，CAS 保证只初始化一次
2. **CAS 插入**：目标桶为空时，CAS 放入新节点，无需加锁
3. **synchronized 处理冲突**：桶不为空时，synchronized 头节点后操作链表/红黑树

CAS 处理了大部分无冲突场景，synchronized 只在冲突时才用，性能接近乐观锁。


