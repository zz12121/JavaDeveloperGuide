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

## 关联知识点

