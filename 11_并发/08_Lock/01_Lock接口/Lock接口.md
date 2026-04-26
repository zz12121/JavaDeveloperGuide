---
title: Lock接口
tags:
  - Java/并发
  - 原理型
module: 08_Lock
created: 2026-04-18
---

# Lock接口（lock/unlock/tryLock/tryLock(timeout)/lockInterruptibly/newCondition）

## 先说结论

`Lock` 接口（`java.util.concurrent.locks.Lock`）提供了比 `synchronized` 更灵活的锁操作：可中断获取、可超时获取、非阻塞尝试获取、公平/非公平选择以及多个条件变量。最常用实现是 `ReentrantLock`。

## 深度解析

### 接口定义

```java
public interface Lock {
    // 获取锁（阻塞直到获取成功）
    void lock();

    // 可中断地获取锁（等待时响应中断）
    void lockInterruptibly() throws InterruptedException;

    // 尝试获取锁（非阻塞，立即返回）
    boolean tryLock();

    // 超时尝试获取锁
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    // 释放锁
    void unlock();

    // 创建新的条件变量
    Condition newCondition();
}
```

### 六个核心方法对比

| 方法 | 阻塞 | 可中断 | 可超时 | 公平保证 |
|------|------|--------|--------|---------|
| `lock()` | ✅ | ❌ | ❌ | 取决于实现 |
| `lockInterruptibly()` | ✅ | ✅ | ❌ | 取决于实现 |
| `tryLock()` | ❌ | N/A | N/A | N/A（非公平） |
| `tryLock(time)` | ✅ | ✅ | ✅ | N/A（非公平） |
| `unlock()` | — | — | — | — |

### 标准使用模式

```java
Lock lock = new ReentrantLock();
lock.lock();  // 或 lock.lockInterruptibly()
try {
    // 临界区
} finally {
    lock.unlock();  // 必须在 finally 中释放
}
```

### 与 synchronized 的能力对比

| 能力 | synchronized | Lock |
|------|-------------|------|
| 获取锁阻塞 | ✅ | ✅ lock() |
| 可中断获取 | ❌ | ✅ lockInterruptibly() |
| 可超时获取 | ❌ | ✅ tryLock(timeout) |
| 非阻塞尝试 | ❌ | ✅ tryLock() |
| 公平锁 | ❌ | ✅ ReentrantLock(true) |
| 多条件变量 | ❌ (只有一个 wait set) | ✅ newCondition() |
| 手动释放 | ❌ (自动) | ✅ unlock() |
| 锁状态查询 | ❌ | ✅ isLocked/isHeldByCurrentThread |

## 易错点/踩坑

- ❌ 忘记在 finally 中 unlock——异常时锁不会释放，导致死锁
- ❌ lock() 在 finally 前可能抛异常——应该 try-finally 包裹整个临界区
- ✅ tryLock() 适合避免死锁：获取多个锁时，如果其中一个失败则释放已获取的锁

## 代码示例

```java
// tryLock 避免死锁
public void transfer(Account from, Account to, int amount) {
    while (true) {
        if (from.lock.tryLock()) {
            try {
                if (to.lock.tryLock()) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        break; // 成功
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
        Thread.yield(); // 获取失败，稍后重试
    }
}

// lockInterruptibly 响应中断
public void doWork() throws InterruptedException {
    lock.lockInterruptibly();  // 等待时可以被中断
    try {
        // 临界区
    } finally {
        lock.unlock();
    }
}
```

## 补充：LockSupport 与 park/unpark 机制

### 地位

`LockSupport` 是 JUC 锁框架的**底层支撑**，AQS 的等待/唤醒、`Condition.await()`/`signal()`、各种 Lock 的 `lock()`/`unlock()` 底层都依赖它。

### 核心 API

```java
public class LockSupport {
    // 阻塞当前线程（permit=0 则 park，permit>0 则立即消费并返回）
    public static void park();
    public static void park(Object blocker);  // blocker 用于调试（Thread.getBlocker）

    // 带超时（纳秒）
    public static void parkNanos(Object blocker, long nanos);

    // 响应中断的 park
    public static void parkUntil(Object blocker, long deadline);

    // 唤醒一个被 park 的线程（+1 permit）
    public static void unpark(Thread thread);
}
```

### park/unpark 对比 Thread.sleep/wait

| 对比维度 | LockSupport.park | Thread.sleep | Object.wait |
|---------|-------------------|--------------|-------------|
| 响应中断 | ✅ 可抛 InterruptedException | ✅ sleep 会中断返回 | ✅ wait 会中断返回 |
| 恢复方式 | unpark() 或中断 | 自动唤醒 | notify/notifyAll |
| 释放锁 | 不涉及（需配合 Lock） | 不释放锁 | 必须持有 monitor |
| 底层实现 | Unsafe.park (OS 层) | Thread.sleep (JVM) | ObjectMonitor |

### 底层原理：permit 令牌机制

每个线程持有一个 `permit`（0 或 1），permit 的行为像一个**先到先得的令牌**：

```
初始：permit = 0 → park() 阻塞
unpark(T)：permit = 1 → T 被唤醒（permit 变为 0）
park()：permit = 1 → 立即消费，park 立即返回
park()：permit = 0 → 阻塞等待
```

**关键特性**：
- permit 最多只有 1 个，多次 unpark 不会累积
- unpark 可以在 park 之前调用（先上票），这是比 `wait/notify` 更灵活的地方

```java
// 先 unpark 再 park → park 立即返回（不阻塞）
LockSupport.unpark(thread);
LockSupport.park();  // permit 已被消耗，不阻塞

// 场景：支持在线程启动前就先唤醒它
Thread t = new Thread(() -> {
    LockSupport.park();   // 等待 permit
    System.out.println("被唤醒");
});
LockSupport.unpark(t);    // 先上票
t.start();                // 线程启动时已被唤醒
```

### park 在 AQS 中的应用

```java
// AQS.acquireQueued 简化
final boolean acquireQueued(Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return interrupted;
            }
            // 重点：shouldParkAfterFailedAcquire 判断后
            // → LockSupport.park(this) 阻塞线程
            if (shouldParkAfterFailedAcquire(p, node))
                LockSupport.park(this);  // 等待被 unpark（来自前驱节点的 release）
            if (Thread.interrupted())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 与 newCondition 的关系

`Condition.await()` 底层调用 `LockSupport.park(this)`，通过不同的 Condition 对象实现**多个独立的等待队列**：

```
Lock锁                    Object monitor
  │                           │
  ├── Condition A → park队列A  ├── wait set（单一）
  └── Condition B → park队列B
```

## 关联知识点
