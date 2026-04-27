
# ConcurrentLinkedQueue

## 核心结论

ConcurrentLinkedQueue 是基于**链表的无界非阻塞队列**，完全使用 CAS 实现线程安全，无锁无阻塞。适合高并发、不需要阻塞等待的场景。

## 深度解析

### 结构

```
ConcurrentLinkedQueue
├── head: volatile Node（队头）
└── tail: volatile Node（队尾）

Node
├── item: volatile E（数据）
└── next: volatile Node（后继指针）
```

### CAS 操作

```java
// CAS 更新节点
static <E> boolean casItem(Node<E> node, E cmp, E val) {
    return UNSAFE.compareAndSwapObject(node, itemOffset, cmp, val);
}

static <E> boolean casNext(Node<E> node, Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(node, nextOffset, cmp, val);
}
```

### offer 源码（入队）

```java
public boolean offer(E e) {
    checkNotNull(e);
    Node<E> newNode = new Node<E>(e);
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p 是尾节点，CAS 挂上新节点
            if (casNext(p, null, newNode)) {
                // CAS 成功，尝试更新 tail（允许失败）
                if (p != t)
                    casTail(t, newNode);
                return true;
            }
            // CAS 失败，重试
        }
        else if (p == q)
            // p 被移除了（自引用），从 head 重新开始
            p = (t != (t = tail)) ? t : head;
        else
            // p.next 不为空，p 不是真正的尾节点，后移
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

### poll 源码（出队）

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            if (item != null && casItem(p, item, null)) {
                // CAS 成功，将 item 设为 null
                if (p != h)
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null; // 队列为空
            }
            else
                p = q; // 后移
        }
    }
}
```

### tail 滞后优化

- tail 不总是指向真正的尾节点（允许一定滞后）
- 每次 offer 只尝试 CAS 更新 tail，失败不影响正确性
- 减少 CAS 操作，提高性能

### 无界队列的注意

- **无界**：不限制队列长度，内存不够时抛 OOM
- **非阻塞**：offer 永远返回 true，poll 空时返回 null
- 不适合需要等待/阻塞的生产者-消费者场景（用 BlockingQueue）

## 易错点与踩坑

### 1. offer 永远返回 true，但不代表成功入队

```java
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();

// ❌ 误以为返回 true = 成功入队
boolean result = queue.offer("item");
System.out.println(result);  // 永远是 true

// ⚠️ 实际上：
// 1. offer 内部用无限循环 CAS，直到成功
// 2. 在极端高并发下，如果内存耗尽，new Node() 会抛 OOM
// 3. 但正常情况下，offer 不会失败

// ✅ 真正的问题是：无界队列 + 内存限制 = OOM
// ✅ 生产环境应该用有界 BlockingQueue
```

### 2. poll 返回 null 的二义性

```java
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();

// ❌ 误以为 null = 队列为空
String item = queue.poll();
if (item == null) {
    // 这里可能是：
    // 1. 队列真的为空
    // 2. 元素本身就是 null（虽然实际不允许，但语义上可能）
}

// ✅ 正确判断队列是否为空
if (queue.isEmpty()) {
    // 队列为空
}

// ⚠️ isEmpty() 也是非精确的（弱一致性）
// 在 isEmpty() 和 poll() 之间，其他线程可能入队/出队
```

### 3. size() 不是一个精确的实时计数

```java
ConcurrentLinkedQueue<Integer> queue = new ConcurrentLinkedQueue<>();

// ❌ size() 遍历整个链表，高并发下不准确
int size = queue.size();  // 遍历中可能有其他线程入队/出队

// ✅ 性能考虑
// - size() 需要遍历链表，O(n) 复杂度
// - 高并发下返回值不精确
// ✅ 如果需要精确计数，用 AtomicInteger 自己维护
```

### 4. tail 不总是指向真正的尾节点

```java
// CLQ 的 offer 实现有"tail 滞后"优化
// tail 允许落后真正的尾节点 1~2 个节点

// ❌ 不能用 tail 来判断队列状态
Node<E> tail = queue.tail;  // 可能不是真正的尾节点

// ✅ 正确做法：遍历时小心
// offer 可能还在入队中，tail 还没更新
// 但最终会收敛到正确的尾节点
```

## 关联知识点
