---
title: cancelAcquire
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# cancelAcquire（取消等待，状态标记，取消后被唤醒处理）

## Q1：cancelAcquire 做了什么？

**A**：处理超时或中断的取消操作：

1. **清除线程引用**：`node.thread = null`
2. **跳过已取消的前驱**：`while (pred.waitStatus > 0) pred = pred.prev`
3. **标记取消**：`node.waitStatus = CANCELLED`
4. **调整指针**：
   - node 是 tail → CAS tail = pred
   - node 在中间 → pred.next 跳过 node
   - 否则 → unpark 后继避免死锁
5. **断开自身**：`node.next = node`

---

## Q2：为什么取消后不立即物理移除节点？

**A**：采用**懒删除**策略：

- 立即遍历清理整个队列成本高，需要加锁
- 其他线程在 `shouldParkAfterFailedAcquire` 中自旋时会自然跳过 CANCELLED 节点
- 取消操作只在必要处修改指针，分摊清理成本

```java
// shouldParkAfterFailedAcquire 中的惰性清理
if (ws > 0) {
    do {
        node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);  // 逐步跳过取消节点
}
```

---

## Q3：什么情况下会触发 cancelAcquire？

**A**：

- `tryLock(timeout)` 超时未获取到锁
- `lockInterruptibly()` 等待时被 `interrupt()` 中断
- `acquireQueued` 中抛出异常（failed=true）

