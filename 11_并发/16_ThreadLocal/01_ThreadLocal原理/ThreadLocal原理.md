
# ThreadLocal原理

## 核心结论

`ThreadLocal` 的核心原理是：**每个线程持有一个 `ThreadLocalMap`，以 `ThreadLocal` 实例为 key、存储值为 value，实现线程间数据隔离**。不同线程访问同一个 `ThreadLocal`，实际操作的是各自线程的 `ThreadLocalMap`，因此互不干扰。

## 深度解析

### 数据结构关系

```
Thread 对象                    ThreadLocalMap                  ThreadLocalMap.Entry
┌──────────────┐              ┌──────────────────┐            ┌──────────────────────┐
│ threadLocals ─┼──────────────│ Entry[] table    │            │ key(ThreadLocal,弱引用) │
│              │              │   [0] Entry ──────┼───────────│ value(强引用)          │
│              │              │   [1] Entry      │            └──────────────────────┘
│              │              │   [2] null       │
│              │              │   ...            │            Entry 继承 WeakReference
└──────────────┘              └──────────────────┘            key 是 ThreadLocal 的弱引用
```

### 核心源码

```java
// ThreadLocal.get()
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = t.threadLocals;       // 获取当前线程的 ThreadLocalMap
    if (map != null) {
        Entry e = map.getEntry(this);           // 以 ThreadLocal 实例为 key 查找
        if (e != null)
            return (T) e.value;
    }
    return setInitialValue();                   // 首次访问，调用 initialValue()
}

// ThreadLocal.set(T value)
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = t.threadLocals;
    if (map != null)
        map.set(this, value);                   // 以 this(ThreadLocal) 为 key
    else
        createMap(t, value);                    // 首次 set，创建 ThreadLocalMap
}

// ThreadLocal.remove()
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);                         // 移除当前 ThreadLocal 对应的 Entry
}
```

### ThreadLocalMap 的 hash 算法

```java
// ThreadLocalMap 的 key 的 hash 值
private final int threadLocalHashCode = nextHashCode();

// 魔数 0x61c88647，黄金比例 * 2^32，使 hash 分布均匀
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 冲突解决：开放寻址法

`ThreadLocalMap` 使用**开放寻址法（线性探测）**解决 hash 冲突，而非 HashMap 的链表/红黑树：

```java
// 简化的 getEntry 逻辑
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);  // 定位
    Entry e = table[i];
    if (e != null && e.get() == key)                         // 直接命中
        return e;
    else
        return getEntryAfterMiss(key, i, e);                 // 线性探测
}
```

### 为什么 key 是弱引用？

```
ThreadLocal 实例          Entry.key            Entry.value
┌─────────────┐           ┌─────────────┐     ┌──────────┐
│  (强引用)    │           │ WeakReference│     │ 强引用    │
│  ref        ├───────────│  referent   │     │ value    │
└─────────────┘           └─────────────┘     └──────────┘
                              ↑ GC 可回收        ↑ GC 不可回收！
```

- **key 为弱引用**：ThreadLocal 外部强引用消失后，GC 可回收 key，避免内存泄漏
- **value 为强引用**：GC 无法回收，如果线程不死亡（线程池复用），value 会一直存在 → **内存泄漏**

## 关联知识点
