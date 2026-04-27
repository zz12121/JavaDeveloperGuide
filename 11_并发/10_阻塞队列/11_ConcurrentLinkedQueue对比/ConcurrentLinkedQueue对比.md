---
title: ConcurrentLinkedQueue对比
tags:
  - Java/并发
  - 对比型
module: 10_阻塞队列
created: 2026-04-27
---

# ConcurrentLinkedQueue对比

## 核心结论

ConcurrentLinkedQueue（CLQ）和 ConcurrentLinkedDeque（CLD）是 JUC 包提供的**无锁非阻塞**队列。完全基于 CAS 实现，不使用锁，吞吐量高，但某些操作（如 size()）是弱一致的。

## 深度解析

### 继承体系

```
Queue
└── AbstractQueue
    └── ConcurrentLinkedQueue    // 单端队列（FIFO）

Deque
└── AbstractDeque
    └── ConcurrentLinkedDeque    // 双端队列
```

### vs BlockingQueue（有锁阻塞队列）

| 维度 | CLQ/CLD | BlockingQueue |
|------|---------|---------------|
| 锁机制 | 无锁（CAS） | 有锁（ReentrantLock） |
| 阻塞 | 非阻塞 | put/take 阻塞等待 |
| 吞吐量 | 极高 | 较高（有锁开销） |
| 适用场景 | 高并发、低延迟 | 生产者-消费者平衡 |
| null | 不支持 | 不支持 |
| size() | O(n) 弱一致 | O(1) 精确 |

### 内部实现

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E> implements Queue<E> {
    private static class Node<E> {
        volatile E item;
        volatile Node<E> next;
    }
    
    private transient volatile Node<E> head;
    private transient volatile Node<E> tail;
}
```

- **CAS 操作**：`Unsafe.getUnsafe().compareAndSwapObject()`
- **HOPS**：tail 和 head 不是精确指向最后一个/第一个节点，而是"跳"过一些节点减少 CAS 操作

### 单端 vs 双端

```java
// ConcurrentLinkedQueue - 单端（FIFO）
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
queue.offer("A");  // 队尾插入
queue.poll();      // 队头移除

// ConcurrentLinkedDeque - 双端
ConcurrentLinkedDeque<String> deque = new ConcurrentLinkedDeque<>();
deque.offerFirst("A");  // 队首插入
deque.offerLast("B");   // 队尾插入
deque.pollFirst();       // 队首移除
deque.pollLast();        // 队尾移除
```

### 弱一致性场景

```java
ConcurrentLinkedQueue<Integer> q = new ConcurrentLinkedQueue<>();
q.add(1);
q.add(2);
q.add(3);

// size() 遍历链表，O(n)
int s = q.size();  // 3，但可能不精确

// contains() 也是 O(n)
boolean has = q.contains(2);

// 高并发下，其他线程可能在遍历过程中修改队列
// 可能看到部分更新
```

## 易错点与踩坑

### 1. size() 是 O(n) 且弱一致
```java
// ❌ 不要在高频路径调用 size()
while (q.size() > 0) {  // 每次都要遍历整个链表！
    process(q.poll());
}

// ✅ 用 isEmpty() 判断
while (!q.isEmpty()) {
    process(q.poll());
}

// ✅ 或者用 AtomicInteger 维护计数
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
count.decrementAndGet();
```

### 2. 迭代器弱一致，可能遗漏或重复元素
```java
// ❌ 迭代过程中队列被修改
Iterator<Integer> it = q.iterator();
while (it.hasNext()) {
    Integer val = it.next();
    q.offer(val * 10);  // 在迭代过程中添加
    // 迭代器可能看不到这个新元素
}

// ✅ 如果需要"快照"，先复制到 List
List<Integer> snapshot = new ArrayList<>(q);
for (Integer val : snapshot) {
    // 安全遍历，不受原队列修改影响
}
```

### 3. 不支持阻塞操作
```java
// ❌ CLQ 没有 put/take，不能阻塞等待
ConcurrentLinkedQueue<String> q = new ConcurrentLinkedQueue<>();
q.put("hello");   // ❌ 编译错误，没有这个方法
q.take();          // ❌ 编译错误

// ✅ 需要自己实现阻塞逻辑
String take() throws InterruptedException {
    while (true) {
        String val = q.poll();
        if (val != null) return val;
        Thread.sleep(10);  // 或用 wait/notify
    }
}

// ✅ 生产者-消费者场景优先用 BlockingQueue
```

### 4. offer 失败不返回 false
```java
// ConcurrentLinkedQueue.offer() 永远返回 true（无界）
q.offer("item");  // 永远成功，不会返回 false

// ⚠️ 与 BlockingQueue.offer() 不同
// BlockingQueue.offer() 满时返回 false
// CLQ 永远不会满（无限增长）

// 如果需要"队列满时拒绝"，用有界队列
```

### 5. 多线程写入时的 ABA 问题
```java
// CLQ 使用 CAS，理论上存在 ABA 问题
// 但 Node 每次操作都会创建新节点，不复用
// 所以 CLQ 实际不受 ABA 问题影响

// 但如果自己实现类似结构，要注意 ABA 问题
// 解决：用 AtomicStampedReference 标记版本
```

### 6. 内存可见性问题
```java
// CLQ 的 volatile 保证可见性
// 但多线程写入后立即读取，可能读到旧值

Thread A:
q.offer("A");
Thread B:
String val = q.poll();  // 可能读不到 A（需要内存屏障）

// ✅ 使用 happens-before 保证
// offer() happens-before poll()（同一对象）
// JDK 已经处理好了，正常使用不需要额外同步
```

## 选型决策

```
需要阻塞等待？
├─ 是 → BlockingQueue
│    ├─ 高吞吐 → LinkedBlockingQueue（双锁）
│    ├─ 固定缓冲 → ArrayBlockingQueue（单锁）
│    └─ 延迟任务 → DelayQueue
└─ 否 → ConcurrentLinkedQueue
     ├─ 单端队列 → ConcurrentLinkedQueue
     └─ 双端队列 → ConcurrentLinkedDeque
```

## 关联知识点

- [[BlockingQueue接口]] - 有锁阻塞队列
- [[ArrayBlockingQueue]] - 有界数组队列
- [[LinkedBlockingQueue]] - 有界链表队列
