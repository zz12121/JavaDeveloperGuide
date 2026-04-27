
# ConcurrentLinkedDeque

## 核心结论

ConcurrentLinkedDeque 是 ConcurrentLinkedQueue 的**双端队列扩展**，支持从两端插入和删除。同样基于 CAS 无锁实现，无界非阻塞。

## 深度解析

### 结构

```
ConcurrentLinkedDeque
├── first: volatile Node（队头）
└── last: volatile Node（队尾）

Node（与 CLQ 不同）
├── prev: volatile Node（前驱指针）
├── item: volatile E（数据）
└── next: volatile Node（后继指针）
```

### API

```java
// 头部操作
offerFirst(e) / pollFirst() / peekFirst()
// 尾部操作
offerLast(e) / pollLast() / peekLast()
// 栈操作（等价于头部操作）
push(e) / pop()
// 普通队列操作
offer(e) / poll() / peek()
```

### 与 ConcurrentLinkedQueue 对比

| 维度 | CLQ | CLD |
|------|-----|-----|
| 方向 | 单端（FIFO） | 双端 |
| 节点 | next 单向 | prev + next 双向 |
| 用途 | 队列 | 队列 + 栈 + 双端 |
| 性能 | 略高（单指针） | 略低（双指针 CAS） |
| CAS 次数 | 每次操作 1~2 次 | 每次操作 2~3 次 |

### 适用场景

1. **工作窃取（Work Stealing）**：ForkJoinPool 中的任务队列
2. **栈结构**：需要 LIFO（后进先出）的并发场景
3. **双端处理**：同一队列两端都有生产者/消费者

## 易错点与踩坑

### 1. 双指针 CAS 导致更高的竞争

```java
// ❌ 与 CLQ 相比，CLD 的每次操作需要 CAS 更多指针
// CLD 节点有 prev + next 两个指针
// 插入/删除需要同时 CAS 两个指针

// 插入操作：CAS next + CAS prev = 2次 CAS
// 删除操作：CAS prev.next + CAS next.prev = 2次 CAS

// ✅ 性能影响：高并发下 CAS 失败重试概率更高
// ✅ 如果只需要单端操作，用 CLQ 性能更好
```

### 2. 作为工作窃取队列时的陷阱

```java
// ForkJoinPool 内部使用 CLD 作为工作队列
// 每个线程从自己队列的尾部取任务（后进先出）

// ❌ 业务代码不要直接操作这个队列
ForkJoinPool pool = ForkJoinPool.common();
ConcurrentLinkedDeque<Task> myQueue = new ConcurrentLinkedDeque<>();

// ⚠️ 多线程同时从同一队列操作会导致竞争
myQueue.offerFirst(task1);  // 线程A
myQueue.pollFirst();        // 线程B（可能抢了A的任务）
```

### 3. 作为栈使用时容易忽略的线程安全问题

```java
// CLD 支持 push/pop，作为栈使用

// ❌ 误以为 pop 是原子的
ConcurrentLinkedDeque<Integer> stack = new ConcurrentLinkedDeque<>();
stack.push(1);
stack.push(2);

// ⚠️ push/pop 不是原子操作
// push = offerFirst
// pop = pollFirst（但可能中途被其他线程干扰）

// ✅ 线程安全的栈：java.util.concurrent.atomic.AtomicReference
AtomicReference<Node> top = new AtomicReference<>();
```

## 关联知识点