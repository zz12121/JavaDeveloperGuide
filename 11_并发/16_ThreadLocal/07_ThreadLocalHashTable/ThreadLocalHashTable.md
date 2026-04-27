
# ThreadLocalHashTable

## 核心结论

`ThreadLocal.ThreadLocalMap` 是 `ThreadLocal` 的底层存储结构，并非真正的 `HashTable`，而是一个**自定义的开放地址哈希表**，使用**线性探测法**解决冲突。Entry 继承 `WeakReference`，key 为弱引用以防止内存泄漏。

## 深度解析

### 数据结构

```
Thread 对象
├── threadLocals       → ThreadLocalMap    （ThreadLocal 的值）
└── inheritableThreadLocals → ThreadLocalMap（InheritableThreadLocal 的值）

ThreadLocalMap
├── Entry[] table        // 底层数组，初始容量 16
├── int size             // 已使用数量
├── int threshold        // 扩容阈值 = capacity * 2/3
└── Entry extends WeakReference<ThreadLocal<?>>
    ├── referent (WeakReference) → ThreadLocal 实例（key）
    └── value                             → 实际存储的值
```

### 为什么不用 HashMap

| 维度 | ThreadLocalMap | HashMap |
|------|---------------|---------|
| key 类型 | WeakReference<ThreadLocal> | 强引用 |
| 冲突解决 | 开放地址（线性探测） | 链表 + 红黑树 |
| 负载因子 | 2/3 | 0.75 |
| 扩容 | 2 倍扩容 | 2 倍扩容 |
| 目的 | 极致精简、少 GC 压力 | 通用高性能 |

选择开放地址法的原因：ThreadLocalMap 的 key 是 ThreadLocal 对象，数量通常很少（几个到几十个），线性探测在数据量小时性能优于链表法，且内存更紧凑。

### hash 计算

```java
// ThreadLocal 的 threadLocalHashCode
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();

// 魔数 0x61c88647（黄金比例 × 2^32）
private static final int HASH_INCREMENT = 0x61c88647;

// hash = hashCode & (table.length - 1)
```

`0x61c88647` 使 hash 值在 2 的幂次大小的数组上分布均匀，减少冲突。

### set() 流程

```
1. 计算 hash = threadLocalHashCode & (len - 1)
2. 遍历 table（线性探测）：
   a. 位置为 null → 创建新 Entry 放入
   b. key 相等（同一 ThreadLocal）→ 更新 value
   c. key 为 null（被 GC 回收）→ replaceStaleEntry() 清理 + 设置
3. size >= threshold → rehash() 扩容
```

### get() 流程

```
1. 计算 hash 定位
2. 线性探测查找：
   a. key 相等 → 返回 value
   b. key 为 null → expungeStaleEntry() 清理后继续找
   c. 到 null → 返回 null
```

### 内存泄漏机制

```
ThreadLocalMap.Entry:
  key = WeakReference<ThreadLocal>  → ThreadLocal 被回收后 key = null
  value = Object（强引用）          → 不会被自动回收

→ ThreadLocal 对象被 GC 后，Entry 的 key 变 null
→ 但 value 仍然存在，造成内存泄漏
→ 必须调用 remove() 清理
```

### 源码关键点

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;  // 强引用，不会被自动回收
    Entry(ThreadLocal<?> k, Object v) {
        super(k);  // key 是弱引用
        value = v;
    }
}
```

## 关联知识点

