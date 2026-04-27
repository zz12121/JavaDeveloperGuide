
# ConcurrentSkipListSet

## 核心结论

ConcurrentSkipListSet 基于 ConcurrentSkipListMap 实现，value 为固定的 Boolean PRESENT。提供线程安全的有序 Set，所有操作平均 O(log n)。

## 深度解析

### 源码

```java
public class ConcurrentSkipListSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {

    private final ConcurrentNavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public ConcurrentSkipListSet() {
        m = new ConcurrentSkipListMap<E, Object>();
    }

    public boolean add(E e) {
        return m.putIfAbsent(e, PRESENT) == null; // key 不存在才添加
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }

    public boolean remove(Object o) {
        return m.remove(o, PRESENT);
    }
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 底层 | ConcurrentSkipListMap |
| 有序 | 自然排序（或 Comparator） |
| 去重 | putIfAbsent |
| 操作复杂度 | O(log n) |
| 范围操作 | 支持（subSet、headSet、tailSet） |
| 线程安全 | CAS + volatile |

### vs CopyOnWriteArraySet

| 维度 | ConcurrentSkipListSet | CopyOnWriteArraySet |
|------|----------------------|---------------------|
| 底层 | 跳表 | 数组 |
| add | O(log n) | O(n) |
| contains | O(log n) | O(n) |
| 有序 | 是 | 否 |
| 写性能 | 好（CAS） | 差（复制数组） |
| 内存 | 索引开销 | 复制开销 |

## 易错点与踩坑

### 1. 与 CopyOnWriteArraySet 的性能选择错误

```java
// ❌ 数据量小时，CSLS 反而可能更慢
// CSLS：O(log n) 查找 + 跳表索引开销
// COWAS：O(n) 查找 + 无索引开销

// 数据量 < 100 时，COWAS 可能更快（常数因子小）
// 数据量 > 100 时，CSLS 明显更快

// ✅ 正确选择：
// 小数据量 + 读多写少 + 不需要有序 → COWAS
// 大数据量 + 需要有序 + 写操作 → CSLS
```

### 2. 依赖 ConcurrentSkipListMap 的线程安全性

```java
// ❌ 误以为可以用视图修改集
ConcurrentSkipListSet<Integer> set = new ConcurrentSkipListSet<>();
set.add(1); set.add(5); set.add(10);

// ⚠️ NavigableSet 视图是弱一致的
NavigableSet<Integer> desc = set.descendingSet();
desc.clear();  // 清空视图

// ⚠️ 原 set 也会被清空！
System.out.println(set.size());  // 0

// ✅ 使用时注意：视图操作会影响原集合
```

### 3. iterator 弱一致性

```java
// ❌ 与 CSLM 一样，迭代器是弱一致的
ConcurrentSkipListSet<String> set = new ConcurrentSkipListSet<>();
set.add("A"); set.add("B"); set.add("C");

Iterator<String> iter = set.iterator();

set.add("D");  // 迭代器可能看不到 D
set.remove("A");  // 迭代器可能仍返回 A

while (iter.hasNext()) {
    System.out.println(iter.next());  // 不确定顺序和内容
}
```

## 关联知识点
