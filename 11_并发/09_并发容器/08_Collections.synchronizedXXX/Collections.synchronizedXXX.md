
# Collections.synchronizedXXX

## 核心结论

`Collections.synchronizedXXX` 是一种**同步包装**方式，将普通集合包装为线程安全集合。所有公共方法用 synchronized 修饰（锁为 mutex 对象）。性能较差，推荐优先使用 JUC 并发容器。

## 深度解析

### 原理

```java
// Collections.synchronizedList
public static <T> List<T> synchronizedList(List<T> list) {
    return new SynchronizedList<>(list);
}

static class SynchronizedList<T> extends SynchronizedCollection<T> implements List<T> {
    final List<T> list;
    SynchronizedList(List<T> list) {
        super(list);
        this.list = list;
    }

    // 所有方法都 synchronized
    public E get(int index) {
        synchronized (mutex) { return list.get(index); }
    }
    public E set(int index, E element) {
        synchronized (mutex) { return list.set(index, element); }
    }
    // ...
}
```

### 锁对象

```java
// 默认 mutex 就是集合本身
SynchronizedCollection(Collection<E> c) {
    this.c = Objects.requireNonNull(c);
    this.mutex = this; // mutex = this
}

// 可指定自定义 mutex
SynchronizedCollection(Collection<E> c, Object mutex) {
    this.c = Objects.requireNonNull(c);
    this.mutex = mutex;
}
```

### 可用的包装方法

```java
Collections.synchronizedList(new ArrayList<>());
Collections.synchronizedSet(new HashSet<>());
Collections.synchronizedMap(new HashMap<>());
Collections.synchronizedSortedMap(new TreeMap<>());
Collections.synchronizedSortedSet(new TreeSet<>());
Collections.synchronizedCollection(new ArrayList<>());
Collections.synchronizedNavigableMap(new TreeMap<>());
Collections.synchronizedNavigableSet(new TreeSet<>());
```

### 复合操作不安全

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// 复合操作不安全！需要外部同步
if (!list.contains("key")) {        // step 1: 加锁检查
    list.add("key");                 // step 2: 加锁添加
} // 两步之间可能被其他线程修改

// 正确写法：手动同步
synchronized (list) {
    if (!list.contains("key")) {
        list.add("key");
    }
}
```

### 迭代器不安全

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// 迭代器必须手动同步！
synchronized (list) {
    for (String s : list) {
        // 安全
    }
}

// 不加锁可能抛 ConcurrentModificationException
```

### vs JUC 并发容器

| 维度 | synchronizedXXX | JUC 并发容器 |
|------|----------------|-------------|
| 锁粒度 | 整个集合一把锁 | 细粒度（桶/段/CAS） |
| 并发度 | 低（串行化） | 高 |
| 复合操作 | 不安全 | 提供原子方法 |
| 迭代器 | 需手动同步 | 安全（COW快照/弱一致） |
| 性能 | 低 | 高 |

### vs WeakHashMap（用途区别）

| 维度 | Collections.synchronizedMap | WeakHashMap |
|------|---------------------------|-------------|
| **目的** | 线程安全 | 自动回收没有强引用的 key |
| **线程安全** | ✅ synchronized 包装 | ❌ 非线程安全（需手动包装） |
| **key 回收** | 永不回收（除非显式 remove） | GC 时无强引用则回收 |
| **典型用途** | 一般并发场景 | 缓存（防止内存泄漏） |
| **可组合** | 可用 `Collections.synchronizedMap(new WeakHashMap<>())` | — |

```java
// WeakHashMap：key 无强引用时自动被 GC 回收
WeakHashMap<Key, CacheEntry> cache = new WeakHashMap<>();
// 当 key 不再被其他地方引用，GC 会自动清理

// 组合使用：线程安全 + 弱引用缓存
Map<Key, CacheEntry> safeCache =
    Collections.synchronizedMap(new WeakHashMap<>());
```

**两者用途互补**：synchronizedXXX 解决并发安全，WeakHashMap 解决内存泄漏。

### vs synchronizedNavigableMap vs ConcurrentSkipListMap

| 维度 | Collections.synchronizedNavigableMap | ConcurrentSkipListMap |
|------|--------------------------------------|----------------------|
| 底层 | TreeMap + synchronized | 跳表 + CAS |
| 线程安全方式 | synchronized（单锁） | CAS + volatile（无锁） |
| 有序 | ✅ 自然排序 | ✅ 自然排序 |
| 范围操作 | ✅ subMap/headMap/tailMap | ✅ subMap/headMap/tailMap |
| Navigable API | ✅ lower/ceiling/higher | ✅ lower/ceiling/higher |
| 并发性能 | 低（串行） | 高（无锁） |
| 推荐场景 | 低并发、简单场景 | 高并发、有序 Map |

> ⚠️ `Collections.synchronizedNavigableMap` 底层是 TreeMap，其 `subMap()` 返回的视图是弱一致性的，迭代时也需要外部同步。

## 易错点与踩坑

### 1. 复合操作需要外部同步（最容易踩的坑）

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// ❌ 复合操作不是原子的！
if (!list.contains("key")) {  // 步骤1：加锁检查
    list.add("key");           // 步骤2：加锁添加
}  // 两步之间其他线程可能已经添加了

// 结果：可能添加重复元素

// ✅ 正确做法：外部手动同步
synchronized (list) {
    if (!list.contains("key")) {
        list.add("key");
    }
}

// ✅ 更好：用并发容器自带的方法
// ConcurrentHashMap: computeIfAbsent
// ConcurrentSkipListMap: putIfAbsent
```

### 2. 迭代时必须手动同步（高频面试点）

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
map.put("A", 1); map.put("B", 2);

// ❌ 迭代时不加锁，抛 ConcurrentModificationException
try {
    for (Map.Entry<String, Integer> entry : map.entrySet()) {
        System.out.println(entry.getKey());
    }
} catch (java.util.ConcurrentModificationException e) {
    System.out.println("迭代时抛异常！");
}

// ✅ 正确做法：在 synchronized 块中迭代
synchronized (map) {
    for (Map.Entry<String, Integer> entry : map.entrySet()) {
        System.out.println(entry.getKey());
    }
}

// ⚠️ 即使加了同步，其他线程可能在迭代期间修改
```

### 3. 默认 mutex 是 this，容易产生死锁

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// ❌ 如果业务代码也在 synchronized(list)，可能导致死锁
synchronized (list) {
    list.add("A");
    // 某处调用了 list 的方法，而这个方法内部又 synchronized(list)
    process(list);  // 可能死锁！
}

void process(List<String> l) {
    synchronized (l) {  // 重入没问题，但如果是不同的锁对象...
        // ...
    }
}

// ✅ 解决方案：使用内部锁对象
List<String> safe = Collections.synchronizedList(
    new ArrayList<>(),
    new Object()  // 自定义 mutex
);
```

### 4. 性能问题被严重低估

```java
// ❌ synchronizedXXX 是单锁，串行化所有操作

// 场景：100 个并发线程同时读
List<String> list = Collections.synchronizedList(new ArrayList<>());

// ❌ 所有读操作都要排队
// 线程1: lock() → 读 → unlock()
// 线程2: lock() → 等待 → 读 → unlock()
// 线程3: lock() → 等待 → ...

// ✅ 正确选择：读多写少用 CopyOnWriteArrayList
// ✅ 正确选择：高并发用 ConcurrentLinkedQueue
// ✅ 正确选择：需要 Map 用 ConcurrentHashMap
```

## 关联知识点

