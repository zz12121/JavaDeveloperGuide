
# 活锁

## Q1：什么是活锁？

**A**：活锁是线程没有阻塞，但一直在做**无效工作**，无法取得进展。线程状态是 RUNNABLE，但实际没有推进业务逻辑。

典型原因：CAS 无限重试、消息互相重发、乐观锁持续冲突。

---

## Q2：活锁和死锁的区别？

**A**：

- **死锁**：线程互相等待对方释放锁，状态 BLOCKED/WAITING，CPU 低
- **活锁**：线程不断重试但不成功，状态 RUNNABLE，CPU 高

死锁的线程是"等死了"，活锁的线程是"忙死了"。

---

## Q3：如何解决活锁？

**A**：

1. **随机退避**：重试前随机等待一段时间，打破同步重试模式
2. **指数退避**：等待时间指数增长（100ms → 200ms → 400ms），避免高频冲突
3. **有界重试**：设定最大重试次数，超过后降级处理
4. **引入优先级**：让高优先级线程先执行，避免平权冲突

生产环境最常用**指数退避 + 有界重试**组合。



> **代码示例：活锁现象与指数退避解决方案**

```java
AtomicInteger counter = new AtomicInteger(0);
int MAX_RETRIES = 5;
int baseDelay = 100;

// ❌ 活锁：CAS 无限重试，没有退避
while (true) {
    int expected = counter.get();
    if (counter.compareAndSet(expected, expected + 1)) break;
    // 高并发下两个线程反复 CAS 失败，CPU 飙高
}

// ✅ 指数退避 + 有界重试
int retries = 0;
while (retries < MAX_RETRIES) {
    int expected = counter.get();
    if (counter.compareAndSet(expected, expected + 1)) break;
    retries++;
    int delay = baseDelay * (1 << retries); // 100 → 200 → 400 → 800 → 1600ms
    Thread.sleep(delay);
}
if (retries >= MAX_RETRIES) { /* 降级处理 */ }
```
