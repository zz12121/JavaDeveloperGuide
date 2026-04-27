
# ThreadLocalHashTable

## Q1：ThreadLocalMap 的底层数据结构是什么？

**A**：

`ThreadLocalMap` 底层是一个**自定义的开放地址哈希表**（不是 `java.util.HashMap`），使用**数组 + 线性探测法**解决冲突。

```java
// Entry 继承 WeakReference，key 为弱引用
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

选择开放地址法而非链表法的原因：ThreadLocalMap 的 key 数量通常很少（几个到几十个），线性探测在小数据量下缓存更友好、内存更紧凑。

---

## Q2：ThreadLocalMap 的 hash 是怎么计算的？

**A**：

```java
private static final int HASH_INCREMENT = 0x61c88647;
private final int threadLocalHashCode = nextHashCode();

// 定位：hash % table.length（实际用位运算）
int index = threadLocalHashCode & (table.length - 1);
```

`0x61c88647` 是**斐波那契散列**的黄金比例乘积（(√5 - 1)/2 × 2³²），使得哈希值在 2 的幂次大小数组上分布非常均匀，有效减少冲突。

---

## Q3：为什么 Entry 的 key 要用 WeakReference？

**A**：

Entry 的 key 是 ThreadLocal 对象本身。如果 key 是强引用，即使外部不再使用 ThreadLocal（`threadLocalRef = null`），ThreadLocalMap 仍然持有强引用，导致 ThreadLocal 对象无法被 GC 回收。

使用 `WeakReference` 后，当外部没有强引用指向 ThreadLocal 时，GC 可以回收它，Entry 的 key 变为 `null`，触发后续清理逻辑。

但 **value 仍然是强引用**，不会自动回收，所以仍然需要手动 `remove()`。

---

## Q4：ThreadLocalMap 的 set/get 方法是怎样工作的？

**A**：

**set 流程**：
1. 用 `threadLocalHashCode & (len-1)` 计算索引
2. 线性探测遍历：找到空位放入新 Entry，找到相同 key 更新 value，找到 key 为 null 则触发清理
3. 超过阈值（2/3 容量）时 rehash 扩容

**get 流程**：
1. 用同样的 hash 计算索引
2. 线性探测查找：key 匹配返回 value，遇到 key 为 null 触发过期清理，遇到 null 返回 null

两个方法都内置了**过期 Entry 的惰性清理**（expungeStaleEntry），在操作过程中顺便清理 key 为 null 的槽位。

---

## Q5：Netty 的 FastThreadLocal 和 JDK ThreadLocal 有什么区别？

**A**：

**JDK ThreadLocal 的问题**：
- 通过 hash 计算定位槽位，hash 冲突时线性探测（O(1) 期望，最坏 O(n)）
- 需要处理 hash 冲突、stale entry 清理等额外逻辑
- ThreadLocalMap 内部有弱引用 GC 机制，存在内存泄漏风险

**FastThreadLocal 的设计**：
- 每个 `FastThreadLocal` 实例创建时，分配一个**全局唯一递增的 index**
- 线程本地存储使用 `InternalThreadLocalMap`，底层是 `Object[] indexedVariables` 数组
- `get()/set()` 直接用 index 访问数组，**O(1) 无哈希冲突**

```java
// FastThreadLocal 核心原理（简化）
public class FastThreadLocal<V> {
    private final int index = InternalThreadLocalMap.nextVariableIndex(); // 全局唯一 index
    
    public V get() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        Object v = threadLocalMap.indexedVariable(index); // 直接数组下标访问
        return (V) v;
    }
    
    public void set(V value) {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        threadLocalMap.setIndexedVariable(index, value); // 直接写入数组
    }
}
```

**核心对比**：

| 维度 | JDK ThreadLocal | FastThreadLocal（Netty） |
|------|----------------|------------------------|
| 存储结构 | ThreadLocalMap（哈希表） | InternalThreadLocalMap（Object数组） |
| 定位方式 | hash + 线性探测 | 唯一index直接访问 |
| 时间复杂度 | O(1) 期望，有冲突时退化 | O(1) 严格，无冲突 |
| 内存 | 开放寻址，数组可能稀疏 | 数组连续，index可能稀疏 |
| 内存泄漏风险 | 弱引用机制 + 需手动remove | 需手动removeAll，否则泄漏 |
| 线程类型要求 | 所有线程 | 最优需配合 FastThreadLocalThread |
| 适用场景 | 通用场景 | Netty高并发网络框架 |

**FastThreadLocal 的前提**：线程需要是 `FastThreadLocalThread`（Netty 自定义线程），否则退化为用 `ThreadLocal` 存储 `InternalThreadLocalMap`，失去性能优势。

**性能差距**：在高并发、大量 ThreadLocal 的场景下，FastThreadLocal 比 JDK ThreadLocal 快约 3~5 倍（减少 hash 计算和探测开销）。

---
