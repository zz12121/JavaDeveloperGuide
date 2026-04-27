
# 死锁避免

## Q1：什么是银行家算法？

**A**：银行家算法是经典的死锁避免算法。核心思想：

1. 系统在分配资源前，先检查分配后是否所有线程仍能完成（安全状态）
2. 如果分配后存在一个安全序列（所有线程都能依次完成），则分配
3. 如果不存在安全序列，则拒绝分配，让线程等待

类似银行家评估贷款风险：不能把所有钱贷给一个人导致其他人没钱。

---

## Q2：死锁预防和死锁避免有什么区别？

**A**：

- **预防**：事先限制线程行为（如按顺序加锁），从根源上破坏必要条件
- **避免**：不事先限制，而是在每次资源分配时动态判断是否安全

Java 开发中基本只用预防，银行家算法更多出现在操作系统理论中。



> **代码示例：死锁预防——按固定顺序加锁**

```java
// ❌ 死锁：两把锁以不同顺序获取
void transfer(Account from, Account to, int amount) {
    synchronized (from) {       // 线程A: lock(A) → lock(B)
        synchronized (to) {     // 线程B: lock(B) → lock(A) → 死锁！
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// ✅ 预防：按固定顺序加锁（破坏"循环等待"条件）
void safeTransfer(Account from, Account to, int amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) { // 始终先锁 ID 小的账户
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

# 解决死锁

## Q1：死锁发生后的恢复方案？

**A**：

1. **重启进程**：最简单直接，但会造成服务中断
2. **中断线程**：`Thread.interrupt()` 唤醒阻塞线程，使其抛异常释放锁
3. **无法强制释放 synchronized**：这是 synchronized 的局限，只能通过中断或等待线程自行退出

线上应急通常是**重启**，然后分析代码从根本上修复。

---

## Q2：编码中如何避免死锁？

**A**：核心实践：

1. **固定加锁顺序**：所有线程按同一顺序获取锁（最有效）
2. **用 tryLock 代替 lock**：设超时，获取失败释放已持有的锁
3. **减小锁粒度**：只锁必要代码，不在持锁时做 IO
4. **开放调用**：不持锁调用外部方法
5. **用并发工具替代**：ConcurrentHashMap、AtomicInteger 等替代 synchronized

---

## Q3：什么是开放调用（open call）？为什么重要？

**A**：开放调用是指**在不持有锁的情况下调用外部方法**。

```java
// 不安全：持锁调用外部方法
synchronized (lock) {
    callback.onComplete(result); // 外部方法可能获取其他锁 → 死锁风险
}

// 安全：开放调用
synchronized (lock) {
    result = compute();
}
callback.onComplete(result); // 锁外调用
```

持锁时调用外部方法是最常见的死锁陷阱之一，因为外部方法内部可能再获取其他锁，形成不可预见的循环等待。