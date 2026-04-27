
# Collections.synchronizedXXX QA

## Q1: Collections.synchronizedMap 和 ConcurrentHashMap 如何选择？

**A:**

| 维度 | Collections.synchronizedMap | ConcurrentHashMap |
|------|---------------------------|-------------------|
| 锁机制 | synchronized（单锁） | CAS + synchronized（桶锁） |
| 并发度 | 低（串行） | 高（桶级并发） |
| get 性能 | O(1) + 锁开销 | O(1)，无锁读 |
| put 性能 | O(1) + 锁开销 | O(1)，细粒度锁 |
| 复合操作 | 需外部同步 | 提供原子方法（computeIfAbsent等） |
| 迭代安全 | 需手动同步 | 安全（弱一致） |
| 适用场景 | 低并发过渡期 | 高并发生产环境 |

**选择建议：**
- 高并发 → `ConcurrentHashMap`
- 低并发、简单场景 → `Collections.synchronizedMap`

---

## Q2: Collections.synchronizedMap 的迭代是否需要加锁？

**A:**

必须加锁。`Collections.synchronizedMap` 的迭代不是线程安全的。

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
map.put("A", 1); map.put("B", 2);

// ❌ 不加锁抛 ConcurrentModificationException
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    // ConcurrentModificationException!
}

// ✅ 必须放在 synchronized 块中
synchronized (map) {
    for (Map.Entry<String, Integer> entry : map.entrySet()) {
        System.out.println(entry.getKey());
    }
}
```

---

## Q3: Collections.synchronizedList 和 Vector 的区别？

**A:**

| 维度 | Collections.synchronizedList | Vector |
|------|----------------------------|--------|
| 实现 | ArrayList + synchronized | 自己实现（早期就有） |
| API | List 接口完整 | List 接口 + 遗留方法 |
| 迭代器 | 需手动同步 | 需手动同步 |
| 扩展性 | 可组合 | 固定 |
| 推荐 | ✅ 现代用法 | 历史遗留（已不推荐） |

```java
// ✅ 推荐
List<String> list1 = Collections.synchronizedList(new ArrayList<>());

// ❌ 不推荐（遗留类）
Vector<String> list2 = new Vector<>();
```

**注意：** 两者底层都是 synchronized，性能相同。推荐 `Collections.synchronizedList` 因为更灵活（可以指定底层 List 类型）。

---

## Q4: Collections.synchronizedMap 的复合操作安全吗？

**A:**

不安全。复合操作需要外部同步。

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());

// ❌ 非原子操作
if (!map.containsKey("key")) {
    map.put("key", 1);
} else {
    map.put("key", map.get("key") + 1);
}
// 两步之间可能被其他线程修改

// ✅ 正确做法
synchronized (map) {
    if (!map.containsKey("key")) {
        map.put("key", 1);
    } else {
        map.put("key", map.get("key") + 1);
    }
}

// ✅ 更好的做法：用 ConcurrentHashMap
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.computeIfAbsent("key", k -> 1);  // 原子操作
```

---

## Q5: Collections.synchronizedMap 如何实现线程安全的缓存？

**A:**

```java
Map<String, Object> cache = Collections.synchronizedMap(new HashMap<>());

// ❌ 普通 get 不安全
public Object getOrLoad(String key) {
    Object value = cache.get(key);
    if (value == null) {
        value = loadFromDB(key);
        cache.put(key, value);  // 可能重复加载
    }
    return value;
}

// ✅ 使用 synchronized 块
public Object getOrLoad(String key) {
    synchronized (cache) {
        Object value = cache.get(key);
        if (value == null) {
            value = loadFromDB(key);
            cache.put(key, value);
        }
        return value;
    }
}

// ✅ 更好的做法：用 ConcurrentHashMap
ConcurrentHashMap<String, Object> cache2 = new ConcurrentHashMap<>();
public Object getOrLoad(String key) {
    return cache2.computeIfAbsent(key, k -> loadFromDB(k));
}
```

---

## Q6: Collections.synchronizedMap 和 Hashtable 的区别？

**A:**

| 维度 | Collections.synchronizedMap | Hashtable |
|------|---------------------------|----------|
| 出现时间 | JDK 1.2 | JDK 1.0 |
| 实现 | 包装器模式 | 直接实现 |
| API | Map 接口完整 | 遗留方法（elements 等） |
| null 支持 | key/value 都可为 null | key/value 都不可为 null |
| 扩展性 | 可组合 | 固定 |
| 推荐 | ✅ 现代用法 | 历史遗留 |

```java
// ✅ 推荐
Map<String, Integer> map1 = Collections.synchronizedMap(new HashMap<>());

// ❌ 不推荐
Hashtable<String, Integer> map2 = new Hashtable<>();

map1.put(null, 1);  // ✅ OK
map2.put(null, 1);  // ❌ NullPointerException
```

---

## Q7: Collections.synchronizedNavigableMap 和 ConcurrentSkipListMap 如何选择？

**A:**

| 维度 | Collections.synchronizedNavigableMap | ConcurrentSkipListMap |
|------|-------------------------------------|----------------------|
| 底层 | TreeMap + synchronized | 跳表 + CAS |
| 锁机制 | 单锁 | 无锁 + 细粒度 |
| 有序性 | ✅ | ✅ |
| 并发性能 | 低 | 高 |
| Navigable API | ✅ | ✅ |
| 适用 | 低并发、有序 Map | 高并发、有序 Map |

```java
// 低并发场景
Map<Integer, String> map1 = Collections.synchronizedNavigableMap(new TreeMap<>());

// 高并发场景
ConcurrentSkipListMap<Integer, String> map2 = new ConcurrentSkipListMap<>();
```

**选择建议：** 高并发 → `ConcurrentSkipListMap`，低并发 → `Collections.synchronizedNavigableMap`。

---

## Q8: Collections.synchronizedXXX 的 mutex 可以自定义吗？

**A:**

可以。通过重载方法指定 mutex 对象。

```java
// ✅ 默认 mutex = this（集合本身）
List<String> list1 = Collections.synchronizedList(new ArrayList<>());

// ✅ 自定义 mutex
final Object lock = new Object();
List<String> list2 = Collections.synchronizedList(new ArrayList<>(), lock);

// ✅ 使用同一个 mutex 的多个集合
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>(), lock);
List<String> list = Collections.synchronizedList(new ArrayList<>(), lock);
```

**使用场景：** 当多个集合需要在业务上保持一致时，使用同一个 mutex。

