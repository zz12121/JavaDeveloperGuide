
# 死锁 vs 活锁 vs 饥饿

## Q1：死锁、活锁、饥饿的区别？

**A**：一句话区分：

- **死锁**：互相等对方释放锁，"等死了"
- **活锁**：不断重试但不成功，"忙死了"
- **饥饿**：永远轮不到执行，"饿死了"

| 对比 | 线程状态 | CPU | 能否自行恢复 |
|------|---------|-----|------------|
| 死锁 | BLOCKED | 低 | ❌ |
| 活锁 | RUNNABLE | 高 | 理论上能 |
| 饥饿 | RUNNABLE | 不定 | 看调度 |

---

## Q2：线上如何快速判断是死锁还是活锁？

**A**：

1. **jstack 看线程状态**：
   - 大量 BLOCKED + "deadlock" → 死锁
   - 全部 RUNNABLE 但无进展 → 可能活锁

2. **看 CPU**：
   - CPU 低 → 死锁（线程在等）
   - CPU 高 → 活锁（线程在忙）

3. **看接口表现**：
   - 完全无响应 → 死锁
   - 有响应但结果不对/超时 → 活锁或饥饿



> **代码示例：死锁、活锁、饥饿的典型代码模式**

```java
Object lockA = new Object(), lockB = new Object();

// 【死锁】两个线程互相等待对方持有的锁
new Thread(() -> { synchronized (lockA) { sleep(); synchronized (lockB) {} } }).start();
new Thread(() -> { synchronized (lockB) { sleep(); synchronized (lockA) {} } }).start();

// 【活锁】CAS 无限重试，线程一直 RUNNABLE 但无进展
AtomicBoolean flag = new AtomicBoolean(false);
new Thread(() -> { while (!flag.compareAndSet(false, true)) { /* 空转 */ } }).start();
new Thread(() -> { while (!flag.compareAndSet(false, true)) { /* 空转 */ } }).start();

// 【饥饿】非公平锁下，新线程总是插队
ReentrantLock lock = new ReentrantLock(false); // 非公平
// 等待线程可能一直无法获取锁（新线程不断插队成功）

