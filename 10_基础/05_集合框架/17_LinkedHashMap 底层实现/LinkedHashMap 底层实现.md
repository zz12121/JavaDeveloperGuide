---
title: LinkedHashMap 底层实现
tags:
  - Java/集合框架
  - 源码型
module: 05_集合框架
created: 2026-04-25
---

# LinkedHashMap 底层实现（HashMap + 双向链表）

## 数据结构

LinkedHashMap 是 HashMap 的子类，在 HashMap 的桶数组结构基础上，增加了**双向链表**来维护元素的**插入顺序**或**访问顺序**。

```
HashMap 结构：
table[]（桶数组）
  ├── [0] → null
  ├── [1] → Node → Node → Node  （链表/红黑树）
  └── ...

LinkedHashMap 结构：
table[] + 双向链表
  ├── 头部：head（最老的节点）↔ tail（最新的节点）
  ├── 每个节点包含：before（前驱）、after（后继）指针
  └── 遍历时按链表顺序遍历，而非数组顺序
```

### 关键字段

```java
public class LinkedHashMap<K, V> extends HashMap<K, V> {
    // 双向链表的头尾节点
    transient LinkedHashMap.Entry<K, V> head;
    transient LinkedHashMap.Entry<K, V> tail;

    // 排序模式：true=访问顺序，false=插入顺序（默认）
    final boolean accessOrder;

    // LinkedHashMap.Entry 继承 HashMap.Node
    static class Entry<K, V> extends HashMap.Node<K, V> {
        LinkedHashMap.Entry<K, V> before;  // 前驱
        LinkedHashMap.Entry<K, V> after;     // 后继

        Entry(int hash, K key, V value, Node<K, V> next) {
            super(hash, key, value, next);
        }
    }
}
```

## HashMap.Node vs LinkedHashMap.Entry

| 字段 | HashMap.Node | LinkedHashMap.Entry |
|------|--------------|---------------------|
| hash | ✅ final int | ✅ 继承 |
| key | ✅ final K | ✅ 继承 |
| value | ✅ V | ✅ 继承 |
| next | ✅ Node<K,V> | ✅ 继承 |
| before | ❌ 无 | ✅ 新增 |
| after | ❌ 无 | ✅ 新增 |

**关键点**：LinkedHashMap.Entry 继承 HashMap.Node，仅新增 before/after 指针，**复用父类的 hash/key/value/next**。

## 排序模式：accessOrder

```java
// 插入顺序（默认）
Map<String, Integer> map1 = new LinkedHashMap<>();
map1.put("a", 1);
map1.put("b", 2);
map1.put("c", 3);
// 遍历顺序：a → b → c

// 访问顺序（用于 LRU）
Map<String, Integer> map2 = new LinkedHashMap<>(16, 0.75f, true);
map2.put("a", 1);
map2.put("b", 2);
map2.put("c", 3);
map2.get("a");  // 访问 "a"，将其移到链表末尾
// 遍历顺序：b → c → a（"a" 最近被访问）
```

| 模式 | accessOrder | 说明 |
|------|-------------|------|
| 插入顺序 | `false`（默认） | 按首次插入顺序遍历 |
| 访问顺序 | `true` | 每次 get/put 都将节点移到末尾 |

## LinkedHashMap 的迭代机制

### 为什么 HashMap 遍历无序？

HashMap 的 table 数组按哈希值散列分布，遍历时按数组下标 + 链表顺序输出，**不反映任何业务意义上的顺序**。

```java
// HashMap 遍历顺序示例
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("z", 1);
hashMap.put("a", 2);
hashMap.put("m", 3);
// 遍历顺序可能是：a → m → z 或其他（依赖哈希分布）
```

### 为什么 LinkedHashMap 遍历有序？

LinkedHashMap 重写了 `newNode()` 等方法，在每次插入/删除时维护双向链表：

```java
// LinkedHashMap 重写的方法
Node<K, V> newNode(int hash, K key, V value, Node<K, V> e) {
    LinkedHashMap.Entry<K, V> p =
        new LinkedHashMap.Entry<>(hash, key, value, e);
    linkNodeLast(p);  // 将新节点添加到链表末尾
    return p;
}

void linkNodeLast(LinkedHashMap.Entry<K, V> p) {
    LinkedHashMap.Entry<K, V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}

// 删除时维护链表
void removeNodeCallback(Node<K, V> node) {
    LinkedHashMap.Entry<K, V> p = (LinkedHashMap.Entry<K, V>) node;
    unlinkNode(p);
}

void unlinkNode(LinkedHashMap.Entry<K, V> p) {
    LinkedHashMap.Entry<K, V> b = p.before;
    LinkedHashMap.Entry<K, V> a = p.after;
    if (b == null) head = a;
    else { b.after = a; p.before = null; }
    if (a == null) tail = b;
    else { b.before = b; a.before = null; }
}
```

遍历时只需从头节点 `head` 沿 `after` 指针依次访问，**保证顺序**。

## LinkedHashMap 遍历 vs HashMap 遍历

```java
// HashMap 遍历（无序）
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("one", 1);
hashMap.put("two", 2);
hashMap.put("three", 3);
// 输出顺序不确定

// LinkedHashMap 遍历（按插入顺序）
Map<String, Integer> linkedHashMap = new LinkedHashMap<>();
linkedHashMap.put("one", 1);
linkedHashMap.put("two", 2);
linkedHashMap.put("three", 3);
for (Map.Entry<String, Integer> entry : linkedHashMap.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
// 输出：one = 1, two = 2, three = 3
```

| 维度 | HashMap | LinkedHashMap |
|------|---------|---------------|
| 数据结构 | 数组 + 链表/红黑树 | HashMap + 双向链表 |
| 遍历顺序 | **无序** | **有序**（插入序/访问序） |
| 内存开销 | 较低 | 额外 2 个指针（before/after） |
| 性能 | 略高（无链表维护） | 略低（维护双向链表） |
| 适用场景 | 只关心 key-value 映射 | 需要有序遍历、LRU 缓存 |

# LRU 实现：removeEldestEntry

## 核心原理

LinkedHashMap 支持自动删除最老节点，通过重写 `removeEldestEntry()` 方法实现：

```java
protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxCapacity;  // 超过容量时删除最老节点
}
```

## 手写 LRUCache（accessOrder=true）

```java
import java.util.*;

class LRUCache<K, V> {
    private final int capacity;
    private final LinkedHashMap<K, V> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        // accessOrder=true：按访问顺序排列
        // removeEldestEntry：当 size() > capacity 时删除最老节点
        this.cache = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > LRUCache.this.capacity;
            }
        };
    }

    public V get(K key) {
        return cache.getOrDefault(key, null);
    }

    public void put(K key, V value) {
        cache.put(key, value);
    }

    public void clear() {
        cache.clear();
    }

    public int size() {
        return cache.size();
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("{");
        cache.forEach((k, v) -> sb.append(k).append("=").append(v).append(", "));
        if (!cache.isEmpty()) {
            sb.setLength(sb.length() - 2);
        }
        return sb.append("}").toString();
    }

    public static void main(String[] args) {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("C", 3);
        System.out.println(cache);  // {A=1, B=2, C=3}

        cache.get("A");  // 访问 A
        System.out.println(cache);  // {B=2, C=3, A=1}  A 移到末尾

        cache.put("D", 4);  // 添加 D，触发删除最老的 B
        System.out.println(cache);  // {C=3, A=1, D=4}
    }
}
```

## 运行结果分析

```
初始状态：{A=1, B=2, C=3}
访问 A 后：{B=2, C=3, A=1}      # A 被移到末尾
添加 D 后：{C=3, A=1, D=4}      # B 被淘汰（最老）
```

## LRU 淘汰策略对比

| 淘汰策略 | 原理 | 适用场景 |
|----------|------|----------|
| LRU（Least Recently Used） | 删除最久未使用的 | 缓存淘汰、热点数据 |
| LFU（Least Frequently Used） | 删除访问频率最低的 | 长期热点数据 |
| FIFO（First In First Out） | 删除最早进入的 | 简单场景 |
| TTL（Time To Live） | 删除超过存活时间的 | 临时数据 |

# LinkedHashMap 源码深度解析

## 继承体系

```
java.util.AbstractMap<K, V>
    └── java.util.HashMap<K, V>
            └── java.util.LinkedHashMap<K, V>
```

## 构造方法

```java
// 默认构造：插入顺序
public LinkedHashMap() {
    super();
    accessOrder = false;
}

// 指定初始容量：插入顺序
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

// 指定初始容量和负载因子：插入顺序
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

// 完整构造：可指定排序模式
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

// 从其他 Map 构造
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super(m);
    accessOrder = false;
}
```

## 关键重写方法

### newNode() - 创建新节点时维护链表

```java
Node<K, V> newNode(int hash, K key, V value, Node<K, V> e) {
    LinkedHashMap.Entry<K, V> p = new LinkedHashMap.Entry<>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

TreeNode<K, V> newTreeNode(int hash, K key, V value, Node<K, V> next) {
    TreeNode<K, V> p = new TreeNode<>(hash, key, value, next);
    linkNodeLast(p);
    return p;
}
```

### afterNodeAccess() - 访问后移到末尾

```java
void afterNodeAccess(Node<K, V> e) {
    LinkedHashMap.Entry<K, V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K, V> p = (LinkedHashMap.Entry<K, V>) e;
        LinkedHashMap.Entry<K, V> b = p.before;
        LinkedHashMap.Entry<K, V> a = p.after;
        p.after = null;
        if (b == null) head = a;
        else b.after = a;
        if (a != null) a.before = b;
        else last = b;
        if (last == null) head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
    }
}
```

**只有在 accessOrder=true 时才移动节点到末尾**。

### afterNodeRemoval() - 删除节点时维护链表

```java
void afterNodeRemoval(Node<K, V> e) {
    LinkedHashMap.Entry<K, V> p = (LinkedHashMap.Entry<K, V>) e;
    LinkedHashMap.Entry<K, V> b = p.before;
    LinkedHashMap.Entry<K, V> a = p.after;
    p.before = p.after = null;
    if (b == null) head = a;
    else b.after = a;
    if (a == null) tail = b;
    else a.before = b;
}
```

## containsValue() 优化

HashMap 的 `containsValue()` 需要遍历所有桶，而 LinkedHashMap 利用链表特性只需遍历一次：

```java
public boolean containsValue(Object value) {
    for (LinkedHashMap.Entry<K, V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == null ? value == null : v.equals(value))
            return true;
    }
    return false;
}
```

| 方法 | HashMap | LinkedHashMap |
|------|---------|---------------|
| containsValue | O(n) 遍历所有桶 | O(n) 遍历链表（更稳定） |

# LinkedHashMap 常用场景

## 场景一：保持插入顺序

```java
// 配置参数、有序属性列表
LinkedHashMap<String, String> config = new LinkedHashMap<>();
config.put("host", "localhost");
config.put("port", "3306");
config.put("db", "mydb");
config.put("user", "root");
config.put("password", "123456");
```

## 场景二：LRU 缓存

```java
// JDK 内置的 LRU 实现（Guava Cache 更强大）
LinkedHashMap<Integer, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 3;
    }
};
```

## 场景三：实现 FIFO 缓存

```java
// accessOrder=false 即为 FIFO
LinkedHashMap<String, Object> fifo = new LinkedHashMap<>(16, 0.75f, false) {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 3;
    }
};
```

## 场景四：实现 MyBatis 一级缓存

```java
// MyBatis 一级缓存就是基于 LinkedHashMap 实现的 LRU 缓存
// 查询结果放入缓存，再次查询时直接返回
```

# LinkedHashMap 线程安全性

## 结论：非线程安全

LinkedHashMap 继承自 HashMap，**不是线程安全的**。多线程环境下可能出现：

1. 扩容时链表结构被破坏
2. 遍历时结构性修改导致 `ConcurrentModificationException`
3. 双向链表指针不一致

## 线程安全方案

```java
// 方案一：Collections.synchronizedMap 包装
Map<String, Integer> syncMap =
    Collections.synchronizedMap(new LinkedHashMap<>());

// 方案二：ConcurrentHashMap + 手动维护顺序（复杂）
// 不推荐，ConcurrentHashMap 不保证顺序

// 方案三：Guava Cache（推荐）
LoadingCache<String, Integer> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .recordStats()
    .build(new CacheLoader<String, Integer>() {
        @Override
        public Integer load(String key) {
            return fetchFromDb(key);
        }
    });
```

# 总结

## LinkedHashMap 核心要点

| 知识点 | 说明 |
|--------|------|
| 继承关系 | extends HashMap |
| 数据结构 | HashMap + 双向链表 |
| Entry 扩展 | 新增 before/after 指针 |
| accessOrder | false=插入顺序，true=访问顺序 |
| LRU 实现 | accessOrder=true + removeEldestEntry |
| 遍历有序 | 依赖 head→tail 链表 |
| 线程安全 | 非线程安全，需外部同步或使用 ConcurrentHashMap |

## HashMap vs LinkedHashMap vs TreeMap

| 维度 | HashMap | LinkedHashMap | TreeMap |
|------|---------|---------------|---------|
| 底层结构 | 数组+链表+红黑树 | HashMap+双向链表 | 红黑树 |
| 遍历顺序 | 无序 | 有序（插入序/访问序） | 排序（Key 比较器） |
| 性能 | O(1) 均摊 | O(1) 均摊（略低） | O(log n) |
| null 支持 | key/value 都可 null | 同 HashMap | key 不可 null |
| 适用场景 | 普通映射 | 有序遍历、LRU | 排序需求 |

## 关联知识点

- HashMap 底层实现
- ConcurrentHashMap 线程安全
- Java 集合框架整体架构
- LRU/LFU 缓存淘汰算法
