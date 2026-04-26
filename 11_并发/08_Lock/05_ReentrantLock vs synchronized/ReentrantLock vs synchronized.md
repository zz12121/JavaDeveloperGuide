---
title: ReentrantLock vs synchronized
tags:
  - Java/并发
  - 对比型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock vs synchronized

## 核心结论

synchronized 是 JVM 内置锁，简洁自动；ReentrantLock 是 API 层面的锁，功能更丰富。**绝大多数场景优先用 synchronized**，需要高级特性时才用 ReentrantLock。

## 深度解析

### 全面对比

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现层面 | JVM 层面（monitorenter/monitorexit） | API 层面（java.util.concurrent） |
| 锁释放 | 自动释放（退出同步块） | 手动 unlock()，必须在 finally 中 |
| 可中断 | 不可中断 | lockInterruptibly() 可中断 |
| 超时获取 | 不支持 | tryLock(timeout) 支持 |
| 公平锁 | 仅非公平 | 支持公平/非公平 |
| 条件变量 | 单一等待队列（wait/notify） | 多个 Condition 独立队列 |
| tryLock | 不支持 | 支持，非阻塞尝试 |
| 锁状态可知 | 不可查询 | getHoldCount/isLocked 等监控方法 |
| 性能 | JDK6 优化后差距不大 | JDK6 优化后差距不大 |
| 异常处理 | 异常自动释放锁 | 需 finally 手动释放 |

### synchronized 优势

1. **简洁**：语法简单，不需要手动释放
2. **自动释放**：异常退出同步块时自动释放锁
3. **JVM 优化**：偏向锁、轻量级锁、锁粗化、锁消除
4. **无内存泄漏风险**：不会忘记释放锁

### ReentrantLock 优势

1. **可中断**：避免死锁，获取锁时可以响应中断
2. **可超时**：tryLock(timeout) 避免无限等待
3. **公平性**：可选公平锁，减少线程饥饿
4. **多条件变量**：精确控制不同等待条件
5. **tryLock**：非阻塞尝试，适合避免死锁的场景

### 选择原则

```
能用 synchronized 就用 synchronized
    ↓ 需要以下特性？
    ├── 可中断 → ReentrantLock
    ├── 可超时 → ReentrantLock
    ├── 公平锁 → ReentrantLock
    ├── 多条件变量 → ReentrantLock
    ├── tryLock → ReentrantLock
    └── 锁状态监控 → ReentrantLock
```

## 代码示例

```java
// synchronized 写法
synchronized (obj) {
    // 临界区
}

// ReentrantLock 等价写法
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}

// ReentrantLock 独有能力：多条件变量
ReentrantLock lock = new ReentrantLock();
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

lock.lock();
try {
    while (queue.isEmpty()) {
        notEmpty.await();  // 在 notFull 条件上等待
    }
    Object item = queue.poll();
    notFull.signal();      // 通知 notFull 条件
} finally {
    lock.unlock();
}
```

## 关联知识点

