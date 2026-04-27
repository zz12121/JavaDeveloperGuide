
# ConcurrentLinkedQueue

## Q1：ConcurrentLinkedQueue 是怎么保证线程安全的？

**A**：完全使用 **CAS（Unsafe.compareAndSwapObject）**，无锁无阻塞：

- `casNext`：CAS 更新节点的 next 指针
- `casItem`：CAS 更新节点的 item（设为 null 表示已出队）
- CAS 失败 → 自旋重试

没有使用 synchronized 或 ReentrantLock。

---

## Q2：为什么 tail 不总是指向真正的尾节点？

**A**：这是**性能优化**：

- 每次 offer 时只"尝试"CAS 更新 tail，失败也不影响正确性
- 下次 offer 时从 tail 开始往后找到真正的尾节点
- 减少了 CAS 操作次数（每次 offer 最多 2 次 CAS）

```
tail 滞后情况：
  head → A → B → C
  tail → B（实际尾是 C）
  下次 offer 从 B.next=C 开始，找到 C 挂新节点
```

---

## Q3：ConcurrentLinkedQueue 有什么缺点？

**A**：

1. **无界**：不限制大小，大量数据可能导致 OOM
2. **非阻塞**：不适合需要等待的生产者-消费者场景
3. **size() 是 O(n)**：需要遍历整个队列
4. **弱一致性**：size 和 isEmpty 不精确
5. **CAS 自旋开销**：高竞争时 CPU 开销大

需要阻塞等待或限流时，使用 BlockingQueue。


