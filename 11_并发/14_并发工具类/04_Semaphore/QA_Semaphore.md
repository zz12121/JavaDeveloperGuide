
# Semaphore

## Q1：Semaphore 的实现原理？

**A**：基于 AQS 共享模式：

1. `Semaphore(int permits)` 将 AQS 的 `state` 设为 permits
2. `acquire()` 调用 `tryAcquireShared(1)`，CAS 递减 state，state < 0 时进入等待队列 park
3. `release()` 调用 `tryReleaseShared(1)`，CAS 递增 state，成功后 `doReleaseShared()` 唤醒等待线程
4. 公平模式额外检查 `hasQueuedPredecessors()`，有排队的线程不插队

---

## Q2：Semaphore(1) 和 ReentrantLock 有什么区别？

**A**：

| 对比 | Semaphore(1) | ReentrantLock |
|------|-------------|---------------|
| 锁获取/释放 | acquire/release | lock/unlock |
| 可重入 | ❌ 不可重入 | ✅ 可重入 |
| 跨线程释放 | ✅ 可以 | ❌ 同一线程释放 |
| Condition | ❌ 不支持 | ✅ 支持 |
| 公平/非公平 | ✅ 都支持 | ✅ 都支持 |

**关键区别**：Semaphore(1) 不可重入，同一线程 acquire 两次会死锁；ReentrantLock 可重入。

---

## Q3：Semaphore 的 release 必须在 finally 中吗？

**A**：是的，和锁一样，`release()` 必须在 `finally` 中：

```java
semaphore.acquire();
try {
    doSomething();
} finally {
    semaphore.release(); // 保证许可释放
}
```

如果不在 finally 中，doSomething() 抛异常后许可不释放，其他线程会永远等待。

---

## Q4：Semaphore 的 acquire 和 release 可以不在同一个线程吗？

**A**：可以。Semaphore 没有绑定线程所有权，一个线程 acquire 的许可可以由另一个线程 release。

这在某些场景下有用（如生产者获取许可代表占用资源，消费者 release 释放），但也容易出 bug，建议保持 acquire/release 在同一方法中配对。

# Semaphore使用场景

## Q1：Semaphore 常见的使用场景有哪些？

**A**：

1. **接口限流**：`tryAcquire()` 快速失败，超过并发数直接返回"系统繁忙"
2. **数据库连接池**：控制最大连接数，Semaphore + BlockingQueue 实现
3. **资源限制**：限制同时打开的文件数、Socket 连接数等
4. **缓存预热**：控制并发初始化的线程数，避免瞬间大量请求打垮数据库

---

## Q2：限流用 Semaphore 好还是用线程池好？

**A**：各有适用场景：

- **Semaphore**：适合控制对**外部资源**的并发访问（数据库、文件、接口），不改变任务执行方式
- **线程池**：适合控制**任务提交**的并发度，管理线程的创建和复用

两者可以组合使用：线程池控制任务线程数，Semaphore 控制对下游服务的并发请求。

---

## Q3：Semaphore 做限流时，tryAcquire 和 acquire 怎么选？

**A**：

- `tryAcquire()`：非阻塞，获取不到立即返回 false。适合**快速失败**场景（接口限流）
- `acquire()`：阻塞，等到许可才继续。适合**必须执行**场景（等待资源可用）

接口限流推荐 `tryAcquire()`，避免请求堆积。

---

```java
// 场景1：接口限流
Semaphore semaphore = new Semaphore(10);  // 最多 10 个并发请求

public void handleRequest() {
    if (semaphore.tryAcquire()) {  // 非阻塞获取
        try {
            processRequest();  // 处理请求
        } finally {
            semaphore.release();
        }
    } else {
        throw new RuntimeException("系统繁忙");
    }
}

// 场景2：简单连接池
Semaphore connections = new Semaphore(5);
connections.acquire();   // 获取连接
try { executeQuery(); }
finally { connections.release(); }
```
