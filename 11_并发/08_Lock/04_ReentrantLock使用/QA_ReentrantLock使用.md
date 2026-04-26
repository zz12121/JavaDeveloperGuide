---
title: ReentrantLock使用
tags:
  - Java/并发
  - 问答
  - 场景型
module: 08_Lock
created: 2026-04-18
---

# ReentrantLock使用

## Q1：ReentrantLock 的标准使用模式是什么？

**A**：

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();       // 获取锁
try {
    // 临界区
} finally {
    lock.unlock();  // 必须在 finally 中释放
}
```

核心要点：
- `lock()` 放在 try 外面，因为 `lock()` 本身不会抛受检异常
- `unlock()` 放在 finally 中，确保任何异常、return 都能释放锁
- lock 几次就要 unlock 几次（可重入计数匹配）

---

## Q2：ReentrantLock 有几种获取锁的方式？分别什么场景使用？

**A**：

| 方法 | 特性 | 适用场景 |
|------|------|---------|
| `lock()` | 阻塞等待，不可中断 | 大多数场景 |
| `tryLock()` | 非阻塞，立即返回 | 避免死锁、降级处理 |
| `tryLock(timeout)` | 超时等待 | 需要超时控制的场景 |
| `lockInterruptibly()` | 可中断等待 | 需要响应中断的场景 |

---

## Q3：使用 ReentrantLock 有哪些常见陷阱？

**A**：

1. **忘记 unlock**：必须在 finally 中释放，否则锁永不释放
2. **unlock 未持有的锁**：抛 `IllegalMonitorStateException`
3. **重入次数不匹配**：lock 3 次 unlock 2 次，锁未完全释放
4. **锁内执行耗时操作**：如 I/O、网络调用，会严重影响并发性能
5. **lock 放在 try 内**：正确做法是 `lock()` 在 try 外面，`unlock()` 在 finally 中


