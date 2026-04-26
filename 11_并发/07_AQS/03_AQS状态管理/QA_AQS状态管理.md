---
title: AQS状态管理
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# AQS状态管理（getState/setState/compareAndSetState）

## Q1：AQS 的 state 字段在不同实现中分别表示什么？

**A**：

| 实现 | state 语义 | 示例 |
|------|-----------|------|
| ReentrantLock | 锁重入次数 | 0=未锁, 3=重入3次 |
| ReadWriteLock | 高16位=读锁, 低16位=写锁 | 0x00020003=2读+3写重入 |
| Semaphore | 可用许可数 | 5=还有5个许可 |
| CountDownLatch | 剩余计数 | 3=还需要3次countDown |
| FutureTask | 任务状态 | NEW→COMPLETING→NORMAL |

state 是通用的 `volatile int`，具体含义完全由子类定义。

---

## Q2：compareAndSetState 和 setState 分别在什么时候用？

**A**：

- **compareAndSetState**：竞争获取锁时使用（首次 tryAcquire），需要原子性保证
- **setState**：已持有锁时使用（重入或释放），当前线程独占，无需 CAS

```java
// ReentrantLock.tryAcquire
if (c == 0) {
    if (compareAndSetState(0, 1))  // 首次获取：CAS
        return true;
} else if (current == owner) {
    setState(c + 1);  // 重入：直接设置（安全，当前线程独占）
    return true;
}
```

---

## Q3：ReadWriteLock 如何用一个 int 表示两种锁？

**A**：通过位运算拆分 32 位 int：

```
state (32位 int)
├── 高16位：读锁计数（最多 65535）
└── 低16位：写锁计数（最多 65535）

读锁计数 = state >>> 16
写锁计数 = state & 0xFFFF
加读锁：state += (1 << 16)   即 state += 65536
加写锁：state += 1
```

## 关联知识点
