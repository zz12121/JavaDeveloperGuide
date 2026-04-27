
# 死锁必要条件

## Q1：产生死锁的四个必要条件？

**A**：
1. **互斥**：资源同时只能被一个线程使用
2. **占有并等待**：线程持有资源的同时等待其他资源
3. **不可抢占**：已获得的资源不能被强制收回
4. **循环等待**：存在 T1→T2→...→Tn→T1 的循环等待链
四个条件**同时满足**才会死锁，破坏任何一个即可预防。

---

## Q2：互斥条件能被破坏吗？

**A**：不能。互斥是锁的基本语义——同一时刻只有一个线程能持有锁。如果破坏互斥条件，就不需要锁了。
所以实际预防死锁只能从后三个条件入手。

---

## Q3：写一个必然产生死锁的代码？

**A**：

```java
Object A = new Object(), B = new Object();

new Thread(() -> {
    synchronized (A) {
        Thread.sleep(50);
        synchronized (B) {} // 等待 B
    }
}).start();

new Thread(() -> {
    synchronized (B) {
        Thread.sleep(50);
        synchronized (A) {} // 等待 A → 死锁
    }
}).start();
```

Thread-1 持有 A 等 B，Thread-2 持有 B 等 A，形成循环等待。


