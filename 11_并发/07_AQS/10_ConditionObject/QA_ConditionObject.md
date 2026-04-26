---
title: ConditionObject
tags:
  - Java/并发
  - 问答
  - 原理型
module: 07_AQS
created: 2026-04-18
---

# ConditionObject（await/signal/signalAll，Object的wait/notify的Lock版本）

## Q1：await 和 signal 的工作原理是什么？

**A**：

**await()**：
1. 加入**条件队列**（单向链表，nextWaiter）
2. **完全释放锁**（fullyRelease，state 归零）
3. **park** 阻塞等待
4. 被 signal 后**转移到同步队列**尾部
5. 重新**获取锁**（恢复之前的重入次数）
6. 从 await() 返回

**signal()**：
1. 从条件队列取出 firstWaiter
2. **转移到同步队列**尾部（transferForSignal）
3. 唤醒该线程去同步队列中争锁

---

## Q2：Condition 和 wait/notify 有什么区别？

**A**：核心区别：

| 维度 | wait/notify | Condition |
|------|------------|-----------|
| 锁 | synchronized | Lock |
| 条件队列数 | 1 个 | 可创建多个 |
| 唤醒精度 | notify 只唤醒1个但不可指定 | signal 精确指定条件 |
| 超时 | wait(ms) | await(ms) |
| 可中断 | 受限于 synchronized | 支持 awaitUninterruptibly |

---

## Q3：signal 后线程立即执行吗？

**A**：不是。signal 只是将线程从条件队列**转移到同步队列**，线程还需在同步队列中排队重新获取锁。获取锁后才会从 await() 返回。

```
await → 条件队列 → signal → 同步队列 → 排队获取锁 → await 返回
```

