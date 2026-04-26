---
title: tryRelease
tags:
  - Java/并发
  - 问答
  - 源码型
module: 07_AQS
created: 2026-04-18
---

# tryRelease（尝试释放锁，唤醒后继节点）

## Q1：tryRelease 的逻辑是什么？什么时候返回 true？

**A**：`tryRelease` 将 state 减 1：

- **state 减 1 后不为 0** → 还有重入 → 返回 false → **不唤醒**后继
- **state 减 1 后为 0** → 锁完全释放 → 清除 owner → 返回 true → **唤醒**后继

```java
int c = getState() - releases;
if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
}
setState(c);
return free;
```

---

## Q2：tryAcquire 用 CAS 但 tryRelease 用 setState，为什么？

**A**：因为调用 `tryRelease` 时，当前线程**一定是锁持有者**（独占模式下），不会有其他线程同时修改 state，所以不需要 CAS。而 `tryAcquire` 是竞争获取，可能多个线程同时 CAS state。

---

## Q3：unparkSuccessor 为什么从 tail 向前找后继？

**A**：因为 `cancelAcquire` 时会将取消节点的 `next` 指针置空，但 `prev` 指针不会。如果从 head.next 找，可能遇到 next 被清空的情况。从 tail 向前找能保证找到最前面的有效等待节点。

## 关联知识点
