---
title: SynchronousQueue
tags:
  - Java/并发
  - 原理型
module: 10_阻塞队列
created: 2026-04-18
---

# SynchronousQueue

## 核心结论

SynchronousQueue 是**不存储元素**的阻塞队列。**put 必须等 take 接收，take 必须等 put 提供**，直接在线程间传递数据。吞吐量高，适合高切换任务。

## 深度解析

### 特性

```java
SynchronousQueue<Integer> queue = new SynchronousQueue<>();

// 线程A
queue.put(1); // 阻塞，直到线程B take

// 线程B
Integer val = queue.take(); // 接收1，put解除阻塞
```

- `size()` 始终为 0
- `peek()` 始终返回 null
- `isEmpty()` 始终返回 true
- 没有内部容量

### 公平/非公平

```java
// 公平模式：FIFO（使用队列）
new SynchronousQueue<>(true);

// 非公平模式：LIFO（使用栈），默认
new SynchronousQueue<>(false);
```

- **公平**：匹配等待最久的线程，避免饥饿
- **非公平**：后到的先匹配，减少线程切换，性能更高

### 内部实现

```
公平模式：TransferQueue（FIFO 队列）
  └── QNode 节点，isData=true(put)/false(take)

非公平模式：TransferStack（LIFO 栈）
  └── SNode 节点，同样区分 put/take
```

核心方法 `transfer()`：put/take 都调用此方法匹配。
- 有等待者 → 直接匹配，交换数据
- 无等待者 → 当前线程入队/入栈等待

### transfer 匹配流程（公平模式）

```
线程A put(1)                    线程B take()
    │                              │
    ▼                              ▼
调用 transfer(1, true)      调用 transfer(null, false)
    │                              │
    ▼                              ▼
检查队列头节点                    检查队列头节点
    │                              │
    ▼                              ▼
无等待者/类型匹配失败          发现 put 等待节点（isData=true）
    │                              │
    ▼                              ▼
创建 QNode(1, true)            匹配成功！
入队等待                          │
    │                              ▼
    │                        交换数据：item 1 → 线程B
    │                        唤醒线程A
    ▼                              │
被线程B唤醒 ◄─────────────────────┘
返回（数据已被取走）
```

**QNode 结构**：
```java
transferer 指向等待的线程
isData    true=数据节点(put), false=请求节点(take)
item      数据（put时非空，take时等待填充）
next      链表指针
```

**匹配原则**：
- put 线程寻找 take 等待者（isData=false）
- take 线程寻找 put 等待者（isData=true）
- 类型匹配 → CAS 交换数据 → unpark 唤醒对方

### 应用场景

```java
// newCachedThreadPool 使用 SynchronousQueue
// 任务提交后直接交给线程，无缓存
Executors.newCachedThreadPool();
// ThreadPoolExecutor(0, Integer.MAX_VALUE, 60s, SynchronousQueue, ...)
```

### vs 其他队列

| 维度 | SynchronousQueue | LinkedBlockingQueue | ArrayBlockingQueue |
|------|-----------------|---------------------|-------------------|
| 容量 | 0 | 有界/无界 | 有界 |
| 存储 | 不存储 | 存储 | 存储 |
| put | 阻塞直到 take | 满才阻塞 | 满才阻塞 |
| take | 阻塞直到 put | 空才阻塞 | 空才阻塞 |
| 吞吐量 | 高（直接传递） | 中 | 低 |
| 用途 | 任务直接交接 | 任务缓冲 | 任务缓冲 |

## 易错点与踩坑

### 1. CachedThreadPool 的线程爆炸问题（核心考点）
```java
// CachedThreadPool 使用 SynchronousQueue
Executors.newCachedThreadPool();
// 等价于 new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>())

// ❌ 危险场景：
// 1. 10000个任务同时提交
// 2. SynchronousQueue.offer() 失败（因为没有等待的 take）
// 3. 线程池创建 10000 个新线程
// 4. 每个线程约 1MB 栈 → 10GB 内存 → OOM

// ✅ 解决方案：给 SynchronousQueue 套一个有界队列的外层
// 或使用自定义 RejectedExecutionHandler
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    0, 100, 60L, SECONDS,
    new SynchronousQueue<>(),
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝时由调用线程执行
);
```

### 2. fair=true 导致的吞吐量下降
```java
// 公平模式（FIFO）：使用 TransferQueue
// 等待最久的 put/take 优先匹配
SynchronousQueue<Integer> fair = new SynchronousQueue<>(true);

// 非公平模式（LIFO）：使用 TransferStack（默认）
SynchronousQueue<Integer> unfair = new SynchronousQueue<>(false);

// 性能对比（非公平是公平的 3~5 倍）
// 但非公平可能导致线程饥饿
```

### 3. put/take 不能并发调用
```java
SynchronousQueue<Integer> q = new SynchronousQueue<>();

// ❌ 死锁：两个线程都等待对方
Thread A = new Thread(() -> {
    try { q.put(1); } catch (InterruptedException e) {}
});
Thread B = new Thread(() -> {
    try { q.take(); } catch (InterruptedException e) {}
});
A.start();
B.start();
// 取决于调度顺序，可能死锁

// ✅ 正确使用：
// 生产者线程只 put，消费者线程只 take
```

### 4. size() 和 isEmpty() 永远返回 0/true
```java
SynchronousQueue<Integer> q = new SynchronousQueue<>();

// ❌ 错误认知
q.put(1);                   // 阻塞，直到另一个线程 take
q.isEmpty();               // 永远是 true
q.size();                  // 永远是 0

// ⚠️ 这是设计意图，不是 bug
// SynchronousQueue 不存储元素，put 和 take 必须配对
```

### 5. peek() 永远返回 null
```java
SynchronousQueue<Integer> q = new SynchronousQueue<>();

// ❌ 错误用法
Integer val = q.peek(); // 永远返回 null，不会阻塞
q.put(1);
Integer val = q.peek(); // 仍然是 null！

// peek() 在 SynchronousQueue 中没有意义
// 必须用 take() 接收数据
```

### 6. 公平模式下的 TransferQueue 匹配顺序
```java
// 公平模式：匹配等待最久的线程
// put线程A → put线程B → take线程C
// A先等待，B后等待，C到达后匹配A

// 这可能导致"反直觉"行为：
// 生产者先 put(1)，再 put(2)
// 消费者 take() → 得到 1（不是2）
// 消费者 take() → 得到 2

// 因为 put(1) 先等待，先被匹配
```

### 7. 与 LinkedTransferQueue 的选择
```java
// SynchronousQueue：put = transfer（阻塞）
// LinkedTransferQueue：put = 入队（不阻塞），transfer = transfer（阻塞）

// ✅ 如果需要缓冲能力，用 LinkedTransferQueue
LinkedTransferQueue<Integer> lq = new LinkedTransferQueue<>();
lq.put(1);  // 不阻塞，直接入队
lq.take();  // 阻塞等待

// ✅ 如果不需要缓冲，用 SynchronousQueue（更简单）
SynchronousQueue<Integer> sq = new SynchronousQueue<>();
sq.put(1);  // 阻塞，直到被 take
```

## 关联知识点

