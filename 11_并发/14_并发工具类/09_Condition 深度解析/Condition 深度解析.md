
# Condition 深度解析

## 核心结论

**Condition** 是 JUC 提供的条件变量，用于线程间的等待/通知机制。它必须与 **Lock** 配合使用（通过 `Lock.newCondition()` 创建），是对 `Object.wait()/notify()` 的增强。

| 对比 | Object.wait/notify | Condition |
|------|---------------------|-----------|
| 锁关联 | 必须配合 synchronized | 必须配合 Lock |
| 精确唤醒 | ❌ signalAll 唤醒所有 | ✅ signal 精确唤醒 |
| 多条件队列 | ❌ 单一等待集 | ✅ 多个 Condition |
| 中断响应 | ❌ 不响应中断 | ✅ 可响应中断 |
| 超时等待 | ❌ 不支持 | ✅ 支持 awaitNanos |

## 核心原理

### AQS + Condition 协作图

```
线程 A (持有锁)
    ↓
await() → 进入 Condition 队列等待
    ↓
释放锁（unlock）
    ↓                    线程 B (等待锁)
    ↓                          ↓
    ↓←←←←←←←←←←←←←←←←←←←←←←←←←←
    │ signal() 唤醒
    ↓
重新获取锁
    ↓
await() 返回
```

### await() 源码流程

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    
    // 1. 当前线程加入 Condition 队列
    Node node = addConditionWaiter();
    
    // 2. 完全释放锁（保存锁状态）
    int savedState = fullyRelease(node);
    
    // 3. 阻塞直到被 signal 或中断
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }
    
    // 4. 重新获取锁
    acquireQueued(node, savedState);
    
    // 5. 处理中断
    if (interrupted)
        thread.interrupted();
}
```

### signal() 源码流程

```java
public final void signal() {
    // 1. 检查是否持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    
    // 2. 从 Condition 队列转移到 AQS 队列
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);  // 遍历找到第一个未取消的节点
}

// 转移到 AQS 队列后，等待获取锁
```

## 关键方法

| 方法 | 说明 |
|------|------|
| `await()` | 阻塞直到被 signal/中断，支持中断响应 |
| `awaitUninterruptibly()` | 阻塞直到被 signal，不响应中断 |
| `awaitNanos(long nanos)` | 阻塞带超时，返回剩余纳秒 |
| `await(long time, TimeUnit unit)` | 阻塞带超时 |
| `awaitUntil(Date deadline)` | 阻塞到指定时间点 |
| `signal()` | 唤醒一个等待线程 |
| `signalAll()` | 唤醒所有等待线程 |

## 实战：生产者-消费者模式

### 标准实现

```java
class BoundedBuffer<E> {
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // 缓冲区不满
    private final Condition notEmpty = lock.newCondition();   // 缓冲区不空
    
    private final E[] items;
    private int count, putIndex, takeIndex;
    
    public BoundedBuffer(int capacity) {
        items = (E[]) new Object[capacity];
    }
    
    // 生产者
    public void put(E item) throws InterruptedException {
        lock.lock();
        try {
            // 缓冲区满，等待消费
            while (count == items.length)
                notFull.await();  // 等待 notFull 信号
            
            items[putIndex] = item;
            putIndex = (putIndex + 1) % items.length;
            count++;
            
            notEmpty.signal();  // 通知消费者有数据
        } finally {
            lock.unlock();
        }
    }
    
    // 消费者
    public E take() throws InterruptedException {
        lock.lock();
        try {
            // 缓冲区空，等待生产
            while (count == 0)
                notEmpty.await();  // 等待 notEmpty 信号
            
            E item = items[takeIndex];
            items[takeIndex] = null;  // 防止内存泄漏
            takeIndex = (takeIndex + 1) % items.length;
            count--;
            
            notFull.signal();  // 通知生产者有空间
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### 对比 Object.wait/notify

```java
// ❌ Object.wait/notify 的问题：所有线程同一个等待集
class BadBuffer<E> {
    private final Object lock = new Object();
    
    public synchronized void put(E item) throws InterruptedException {
        while (isFull())
            lock.wait();  // 所有等待线程都会醒来，需要再检查
        
        doPut(item);
        lock.notify();  // 可能唤醒另一个生产者
    }
    
    public synchronized E take() throws InterruptedException {
        while (isEmpty())
            lock.wait();
        
        E item = doTake();
        lock.notify();  // 可能唤醒另一个消费者
        return item;
    }
}

// ✅ Condition 的优势：精确唤醒
class GoodBuffer<E> {
    // 不同的 Condition 等待不同条件
    notFull.await();   // 只有生产者在这里等
    notEmpty.signal(); // 只唤醒生产者
    
    notEmpty.await();  // 只有消费者在这里等
    notFull.signal();  // 只唤醒消费者
}
```

## 易错点与踩坑

### 1. signal 在 lock 之外调用

```java
// ❌ 错误：signal 时没有持有锁
Condition condition = lock.newCondition();

Thread t1 = new Thread(() -> {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock();
    }
});

Thread t2 = new Thread(() -> {
    // ⚠️ 没有获取锁就调用 signal
    condition.signal();  // IllegalMonitorStateException
});

t1.start();
Thread.sleep(100);
t2.start();

// ✅ 正确：signal 前必须获取锁
Thread t2 = new Thread(() -> {
    lock.lock();
    try {
        condition.signal();  // OK
    } finally {
        lock.unlock();
    }
});
```

### 2. signalAll 导致惊群效应

```java
// ❌ 误以为 signalAll 比 signal 更好
// 实际上：
// - signal: 唤醒一个，精确通知
// - signalAll: 唤醒所有，性能差（大量线程竞争锁）

// ✅ 大部分场景用 signal
// ✅ signalAll 只用于：
// 1. 不知道哪个线程应该继续
// 2. 所有等待线程都可以继续
// 3. 需要唤醒所有等待者（如关闭操作）
```

### 3. await 前没检查条件

```java
// ❌ 虚假唤醒：没有用 while 检查条件
lock.lock();
try {
    // ❌ 可能被 signalAll 唤醒，但条件可能还没满足
    if (count == 0)  // 应该用 while！
        condition.await();
} finally {
    lock.unlock();
}

// ✅ 正确做法：始终用 while 检查条件
lock.lock();
try {
    while (count == 0)  // ✅
        condition.await();
} finally {
    lock.unlock();
}
```

### 4. 异常导致锁未释放

```java
// ❌ finally 中忘记释放锁
lock.lock();
try {
    doSomething();
    if (someCondition)
        throw new RuntimeException();  // 锁不会被释放！
} catch (Exception e) {
    // 处理异常
} finally {
    // 忘记 unlock()！其他线程永久等待
}

// ✅ 正确做法：确保 finally 释放锁
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();  // ✅
}
```

## 应用场景

| 场景 | Condition 优势 |
|------|--------------|
| 生产者-消费者 | 精确唤醒，不同条件队列 |
| 读线程-写线程协调 | 读写分离等待 |
| 多阶段任务 | 每个阶段独立 Condition |
| 线程池任务协调 | 任务完成通知 |
| 连接池管理 | 获取/归还连接 |

## 关联知识点

- [[ReentrantLock]]
- [[ReentrantReadWriteLock]]
- [[CyclicBarrier]]（内部使用 Condition）
- [[AQS]]（Condition 队列与 AQS 队列的转换）
