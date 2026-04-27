
# QA_Condition 深度解析

## Q1: Condition 和 Object.wait/notify 的核心区别？

**答：**

| 区别 | Object.wait/notify | Condition |
|------|-------------------|----------|
| 锁 | 必须配合 synchronized | 必须配合 Lock |
| 条件队列 | 单一等待集 | 多个条件队列 |
| 精确唤醒 | ❌ 只能 notifyAll | ✅ signal 单个 |
| 中断响应 | ❌ 不响应 | ✅ await 可响应中断 |
| 超时 | ❌ 不支持 | ✅ 支持 awaitNanos |
| 调用位置 | 必须在同步块 | 必须在 lock 块内 |

**Condition 优势**：
```java
// Object.wait/notify：所有线程同一个等待集
synchronized(obj) {
    while (condition not met) {
        obj.wait();  // 所有线程都在这里等
    }
}
obj.notifyAll();  // 唤醒所有，可能惊群

// Condition：多个独立等待集
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();   // 队列不满
Condition notEmpty = lock.newCondition();  // 队列不空

notFull.await();   // 生产者在这里等
notEmpty.signal(); // 只唤醒消费者
```

---

## Q2: Condition 的 await/signal 原理是什么？

**答：**

**await() 原理**：
```
1. 当前线程加入 Condition 队列（单向链表）
2. 完全释放锁（保存锁状态到 AQS）
3. LockSupport.park() 阻塞
4. 被 signal 后，重新获取锁
5. await() 返回
```

**signal() 原理**：
```
1. 检查是否持有锁
2. 从 Condition 队列取出第一个节点
3. 转移到 AQS 同步队列
4. LockSupport.unpark() 唤醒
5. 线程重新竞争锁
```

**AQS + Condition 队列转换**：
```
Condition 队列                    AQS 队列
+--------+  signal()  +--------+  acquireQueued()  +--------+
| Node   | ---------> | Node   | ----------------> | Node   |
|waiter=thread|      |next    |                   |status  |
+--------+            +--------+                   +--------+
```

---

## Q3: 为什么 await() 必须用 while 循环？

**答：**

**虚假唤醒（Spurious Wakeup）**：

JVM 规范允许 `LockSupport.park()` 在没有调用 `unpark()` 的情况下返回。这是为了提高线程调度效率，但会导致线程"莫名其妙"醒来。

```java
// ❌ 错误：用 if 判断
lock.lock();
try {
    if (count == 0) {  // 可能虚假唤醒，醒来时 count 可能还是 0
        condition.await();  // 但代码继续往下走了
    }
    // 此时可能 NPE 或其他错误
} finally {
    lock.unlock();
}

// ✅ 正确：用 while 循环
lock.lock();
try {
    while (count == 0) {  // 醒来后再次检查条件
        condition.await();
    }
    // 此时 count > 0，可以安全使用
} finally {
    lock.unlock();
}
```

**signalAll 场景**：
```java
// 如果用 if，多个线程被唤醒后都会继续执行
// 但资源可能只够一个线程使用
// while 循环确保每个线程醒来后都重新检查条件
```

---

## Q4: signal() 和 signalAll() 怎么选？

**答：**

| 场景 | 推荐 | 原因 |
|------|------|------|
| 只有一个线程应该继续 | signal | 精确唤醒，性能好 |
| 不知道哪个线程应该继续 | signalAll | 所有线程都检查 |
| 关闭/终止操作 | signalAll | 唤醒所有等待线程 |
| 资源分配（多个消费者） | signalAll | 可能多个消费者都能处理 |

```java
// ✅ signal 场景：生产者-消费者
notFull.await();   // 生产者等不满
notEmpty.signal(); // 只唤醒一个消费者

// ✅ signalAll 场景：关闭所有等待线程
running = false;
allConditions.signalAll();  // 唤醒所有等待线程

// ❌ signalAll 滥用：性能差
// 100个线程等待，signalAll 唤醒100个
// 99个竞争锁后发现条件不满足，又回去等待
// 大量无效的线程切换
```

---

## Q5: await() 等待时锁被释放，线程安全吗？

**答：**

**完全安全**。Condition 的 await() 实现：

```java
public final void await() {
    // 1. 加入 Condition 队列
    Node node = addConditionWaiter();

    // 2. 完全释放锁（关键！）
    int savedState = fullyRelease(node);  // 保存状态并释放

    // 3. 阻塞
    LockSupport.park(this);

    // 4. 重新获取锁
    acquireQueued(node, savedState);
}
```

**为什么安全**：
1. await() 释放锁时，`fullyRelease()` 会完全释放（包括可重入次数）
2. signal() 被调用后，线程被唤醒并重新获取锁
3. 获取锁后才能从 await() 返回
4. 其他线程在 await() 期间可以获取锁

---

## Q6: Condition 有哪些变体方法？

**答：**

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `await()` | 等待，支持中断 | void |
| `awaitUninterruptibly()` | 等待，忽略中断 | void |
| `awaitNanos(long nanos)` | 等待，最长nanos纳秒 | 剩余纳秒 |
| `await(long time, TimeUnit unit)` | 等待，最长指定时间 | boolean |
| `awaitUntil(Date deadline)` | 等待，直到指定时间 | boolean |
| `signal()` | 唤醒一个等待线程 | void |
| `signalAll()` | 唤醒所有等待线程 | void |

**使用示例**：
```java
// 带超时的等待
boolean gotSignal = condition.await(5, TimeUnit.SECONDS);
if (!gotSignal) {
    // 超时处理
}

// 等待到指定时间
boolean gotSignal = condition.awaitUntil(deadline);
if (!gotSignal) {
    // 超时处理
}

// 不响应中断的等待
condition.awaitUninterruptibly();
```

---

## Q7: 生产者-消费者模式中 Condition 如何避免死锁？

**答：**

**常见死锁模式**：

```java
// ❌ 死锁：signal 时没有释放锁（不可能，因为 signal 在 lock 内）
// ❌ 死锁：生产者等 notFull，消费者等 notEmpty，但永远不 signal

// ❌ 死锁：两个不同的锁互相等待
Lock lock1 = new ReentrantLock();
Lock lock2 = new ReentrantLock();
Condition cond1 = lock1.newCondition();
Condition cond2 = lock2.newCondition();

// 线程 A: lock1 -> await(cond1) -> 等待 lock2
// 线程 B: lock2 -> await(cond2) -> 等待 lock1
// → 死锁！
```

**安全模式**：
```java
// ✅ 正确做法：始终在 finally 中 unlock
public void put(E item) {
    lock.lock();
    try {
        while (count == items.length) {
            notFull.await();  // 可能被中断
        }
        items[putIndex] = item;
        putIndex = (putIndex + 1) % items.length;
        count++;
        notEmpty.signal();
    } finally {
        lock.unlock();  // 确保释放
    }
}

// ✅ 正确做法：await 超时处理
boolean notFullSignaled = notFull.await(10, TimeUnit.SECONDS);
if (!notFullSignaled) {
    throw new RuntimeException("生产者等待超时");
}
```

---

## Q8: Condition 的使用场景有哪些？

**答：**

**1. 生产者-消费者模式**
```java
// 缓存队列、空池管理
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
```

**2. 线程池任务协调**
```java
// 等待任务完成
Condition taskComplete = lock.newCondition();
while (!task.isDone()) {
    taskComplete.await();
}
```

**3. 连接池管理**
```java
// 等待可用连接
Condition connectionAvailable = lock.newCondition();
while (pool.isEmpty()) {
    connectionAvailable.await();
}
Connection conn = pool.getConnection();
```

**4. 限流器**
```java
// 等待令牌
Condition permitAvailable = lock.newCondition();
while (permits <= 0) {
    permitAvailable.await();
}
permits--;
```

**5. 多阶段任务**
```java
// 每阶段完成后 signal 下一阶段
Condition phase1Complete = lock.newCondition();
Condition phase2Complete = lock.newCondition();
// ...
```
