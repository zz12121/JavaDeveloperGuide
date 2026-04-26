---
title: LinkedBlockingQueue
tags:
  - Java/并发
  - 问答
  - 源码型
module: 10_阻塞队列
created: 2026-04-18
---

# LinkedBlockingQueue

## Q1：LinkedBlockingQueue 的双锁设计有什么优势？

**A**：

- **putLock**：保护入队操作（last 节点和 count）
- **takeLock**：保护出队操作（head 节点和 count）

put 和 take 使用不同的锁，可以同时进行，互不阻塞。ArrayBlockingQueue 只有一把锁，put/take 串行执行。

在高并发场景下，LBQ 的吞吐量明显优于 ABQ。

---

## Q2：LinkedBlockingQueue 的 count 为什么用 AtomicInteger？

**A**：因为 put 和 take 用不同的锁，count 需要在两个锁之外被访问。AtomicInteger 通过 CAS 保证 count 的原子更新，避免了锁竞争。

put 时 `count.getAndIncrement()`，take 时 `count.getAndDecrement()`。

---

## Q3：newFixedThreadPool 用 LinkedBlockingQueue 有什么风险？

**A**：`Executors.newFixedThreadPool` 使用**默认容量的 LBQ**（Integer.MAX_VALUE）。

当核心线程都在忙时，新任务全部堆积在队列中，队列无限增长导致 OOM。

解决方案：手动创建 ThreadPoolExecutor，指定有界队列容量。

---
```java
// LinkedBlockingQueue：双锁设计
// putLock 保护入队，takeLock 保护出队
// count 用 AtomicInteger 保证原子性
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);  // 有界

queue.put("a");   // putLock.lock() → last.next = node → putLock.unlock()
String e = queue.take();  // takeLock.lock() → head = head.next → takeLock.unlock()

// put 和 take 可以同时进行，互不阻塞（吞吐量优于 ArrayBlockingQueue）
```


