
# ConcurrentLinkedDeque QA

## Q1: ConcurrentLinkedDeque 和 ConcurrentLinkedQueue 的区别？

**A:**

| 维度 | ConcurrentLinkedQueue | ConcurrentLinkedDeque |
|------|----------------------|---------------------|
| 方向 | 单端（FIFO） | 双端 |
| 节点结构 | `next` 单向 | `prev` + `next` 双向 |
| CAS 次数 | 1~2 次/操作 | 2~3 次/操作 |
| API | offer/poll/peek | offerFirst/Last/pollFirst/Last + push/pop |
| 性能 | 略高 | 略低 |
| 典型应用 | 普通队列 | 工作窃取、栈 |

**选择建议：** 单端需求用 CLQ，双端需求用 CLD。

---

## Q2: ConcurrentLinkedDeque 可以作为工作窃取队列吗？

**A:**

可以。`ForkJoinPool` 内部正是用 `ConcurrentLinkedDeque` 作为工作队列。

```java
// ForkJoinPool 内部结构
class ForkJoinWorkerThread extends Thread {
    final ForkJoinPool pool;
    final ForkJoinPool.WorkQueue workQueue;  // 内部使用 CLD
}
```

**工作窃取模式：**
```java
// 每个线程从自己队列的尾部取任务（后进先出）
// 其他空闲线程从队列头部偷任务（先进先出）

// 本线程：pollLast() 从尾部取
// 偷取线程：pollFirst() 从头部取
```

---

## Q3: ConcurrentLinkedDeque 作为栈使用时需要注意什么？

**A:**

`CLD` 支持 `push/pop` 操作（等价于 `offerFirst/pollFirst`），但这不是原子的栈操作。

```java
ConcurrentLinkedDeque<Integer> stack = new ConcurrentLinkedDeque<>();

// ⚠️ 多线程同时操作会有竞争
stack.push(1);
stack.push(2);
Integer val = stack.pop();  // 返回 2

// ✅ 建议：用 AtomicReference 实现无锁栈
AtomicReference<Node> top = new AtomicReference<>();

// ✅ 或者用 synchronized 包装
Deque<Integer> safeStack = Collections.synchronizedDeque(new ArrayDeque<>());
```

---

## Q4: ConcurrentLinkedDeque 支持阻塞操作吗？

**A:**

不支持。`CLD` 是非阻塞队列，没有 `take()` 方法会阻塞。

```java
ConcurrentLinkedDeque<String> deque = new ConcurrentLinkedDeque<>();

deque.pollFirst();  // 空队列返回 null，不会阻塞
deque.pollLast();   // 空队列返回 null，不会阻塞

// ✅ 如果需要阻塞：用 LinkedBlockingDeque
LinkedBlockingDeque<String> blockingDeque = new LinkedBlockingDeque<>();
blockingDeque.takeFirst();  // 空时会阻塞
```

---

## Q5: ConcurrentLinkedDeque 的迭代器是线程安全的吗？

**A:**

是线程安全的，但迭代器本身不支持并发修改。

```java
ConcurrentLinkedDeque<Integer> deque = new ConcurrentLinkedDeque<>();
deque.add(1); deque.add(2); deque.add(3);

// ✅ 迭代器可以安全创建
Iterator<Integer> iter = deque.iterator();

// ⚠️ 迭代过程中修改：
deque.add(4);  // 可以，但迭代器可能看不到

// ❌ 不支持迭代时删除
while (iter.hasNext()) {
    iter.next();
    iter.remove();  // ❌ UnsupportedOperationException
}
```

---

## Q6: 什么时候用 ConcurrentLinkedDeque 而不是 ConcurrentLinkedQueue？

**A:**

| 场景 | 推荐 | 原因 |
|------|------|------|
| 普通 FIFO 队列 | CLQ | 单端更高效 |
| LIFO 栈 | CLD | 双端支持 push/pop |
| 工作窃取 | CLD | ForkJoinPool 的选择 |
| 双端处理 | CLD | 两端都可操作 |
| 高并发单端 | CLQ | CAS 次数更少 |

