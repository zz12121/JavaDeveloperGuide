
# 死锁预防

## Q1：如何预防死锁？

**A**：破坏四个必要条件之一：

1. **破坏循环等待**：按固定顺序获取锁（最常用）
2. **破坏占有并等待**：一次性获取所有需要的锁
3. **破坏不可抢占**：使用 tryLock + 超时，获取失败则释放已持有的锁
4. **减少锁粒度**：尽量缩短持锁时间和锁范围

实际开发中最推荐**按固定顺序加锁**。

---

## Q2：转账场景如何避免死锁？

**A**：经典方案是**按账户 hashCode 排序**加锁：

```java
Account first = from.id < to.id ? from : to;
Account second = from.id < to.id ? to : from;
synchronized (first) {
    synchronized (second) {
        from.debit(amount);
        to.credit(amount);
    }
}
```

所有线程都按同样的顺序获取锁，不会形成循环等待。

---

## Q3：tryLock 为什么能预防死锁？

**A**：tryLock 尝试获取锁，如果获取不到不会一直阻塞（可设超时），而是返回 false。获取失败后可以释放已持有的锁并重试，打破了"不可抢占"条件——线程不会永远占着锁不放。


