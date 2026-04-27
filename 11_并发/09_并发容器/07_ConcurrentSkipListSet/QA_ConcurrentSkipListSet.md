
# ConcurrentSkipListSet QA

## Q1: ConcurrentSkipListSet 和 ConcurrentSkipListMap 是什么关系？

**A:**

`ConcurrentSkipListSet` 基于 `ConcurrentSkipListMap` 实现，value 使用固定的 `PRESENT` 对象。

```java
public class ConcurrentSkipListSet<E> extends AbstractSet<E>
    implements NavigableSet<E> {

    private final ConcurrentNavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public boolean add(E e) {
        return m.putIfAbsent(e, PRESENT) == null;
    }

    public boolean remove(Object o) {
        return m.remove(o, PRESENT);
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }
}
```

**特点：**
- Set 的所有操作映射到 Map 操作
- value 固定为 PRESENT，不占用额外空间（内存忽略不计）

---

## Q2: ConcurrentSkipListSet 的元素是否有序？

**A:**

是的。`ConcurrentSkipListSet` 按自然顺序（或指定的 Comparator）排序。

```java
ConcurrentSkipListSet<Integer> set = new ConcurrentSkipListSet<>();
set.add(5);
set.add(1);
set.add(3);

// 遍历顺序：1, 3, 5（有序）
set.forEach(System.out::println);
```

**注意：** 有序是指按元素本身的值排序，不是按插入顺序。

---

## Q3: ConcurrentSkipListSet 支持 NavigableSet API 吗？

**A:**

支持。`ConcurrentSkipListSet` 实现完整的 `NavigableSet` 接口。

```java
ConcurrentSkipListSet<Integer> set = new ConcurrentSkipListSet<>();
set.add(10); set.add(20); set.add(30); set.add(40); set.add(50);

// lower/ceil/higher
set.lower(30);      // 20（严格 < 30）
set.floor(30);      // 30（<= 30）
set.ceiling(30);    // 30（>= 30）
set.higher(30);     // 40（严格 > 30）

// 边界
set.first();        // 10（最小）
set.last();         // 50（最大）

// 子集
set.subSet(20, true, 40, false);  // [20, 40) → 20, 30
set.headSet(30, true);            // <= 30
set.tailSet(30, false);           // > 30
```

---

## Q4: ConcurrentSkipListSet 和 CopyOnWriteArraySet 如何选择？

**A:**

| 维度 | ConcurrentSkipListSet | CopyOnWriteArraySet |
|------|----------------------|---------------------|
| 有序性 | ✅ 按自然顺序 | ❌ 无序 |
| add 复杂度 | O(log n) | O(n) |
| contains 复杂度 | O(log n) | O(n) |
| 写性能 | 好（CAS） | 差（复制数组） |
| 迭代一致性 | 弱一致 | 快照 |
| 适用数据量 | 大 | 小 |

**选择建议：**
- 需要有序 → `ConcurrentSkipListSet`
- 小数据量 + 读多写少 + 无需有序 → `CopyOnWriteArraySet`

---

## Q5: ConcurrentSkipListSet 是线程安全的吗？

**A:**

是的。所有操作都是线程安全的，通过 CAS + volatile 实现。

```java
ConcurrentSkipListSet<String> set = new ConcurrentSkipListSet<>();

// 多线程安全
set.add("A");      // CAS
set.remove("B");    // CAS
set.contains("C"); // volatile 读
set.size();         // 近似准确
```

**注意：** `size()` 不是精确的实时计数，与 `ConcurrentHashMap` 类似。

---

## Q6: ConcurrentSkipListSet 可以用于排行榜吗？

**A:**

可以。`ConcurrentSkipListSet` 的 Navigable API 非常适合排行榜场景。

```java
ConcurrentSkipListSet<Long> scores = new ConcurrentSkipListSet<>();
scores.add(100L);
scores.add(95L);
scores.add(98L);
scores.add(88L);

// 获取前10名
Iterator<Long> top10 = scores.descendingIterator();
int rank = 1;
while (top10.hasNext() && rank <= 10) {
    System.out.println("第" + rank + "名: " + top10.next());
    rank++;
}

// 查询某玩家的排名
Long myScore = 96L;
Long higher = scores.higher(myScore);  // 98（排名更高）
int myRank = scores.tailSet(myScore, true).size();  // 自己的排名
```

---

## Q7: ConcurrentSkipListSet 的迭代器是否支持并发修改？

**A:**

支持并发读取，但不支持迭代期间的修改（弱一致）。

```java
ConcurrentSkipListSet<Integer> set = new ConcurrentSkipListSet<>();
set.add(1); set.add(2); set.add(3);

Iterator<Integer> iter = set.iterator();

// ✅ 可以在迭代期间在其他线程添加元素
set.add(4);  // 迭代器可能看不到

// ❌ 不支持迭代器的 remove
while (iter.hasNext()) {
    iter.next();
    iter.remove();  // UnsupportedOperationException
}
```

