---
title: ConcurrentLinkedQueue
tags:
  - Java/并发
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

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

## 关联知识点
