
# ConcurrentSkipListMap

## 核心结论

ConcurrentSkipListMap 是**跳表（Skip List）** 实现的并发有序 Map，是 TreeMap 的并发安全版本。所有操作平均 O(log n)，通过 CAS + volatile 实现线程安全。

## 深度解析

### 跳表结构

```
Level 4:  [1] ----------------------------------------→ [9]
Level 3:  [1] ----------→ [5] ----------→ [9]
Level 2:  [1] ----→ [3] ----→ [5] ----→ [7] ----→ [9]
Level 1:  [1] → [2] → [3] → [4] → [5] → [6] → [7] → [8] → [9]
```

- 底层是有序链表（Level 1）
- 上层是"快速通道"索引，通过概率随机生成
- 查找时从最高层开始，逐层下降，类似二分查找
- 平均查找复杂度 O(log n)

### 内部结构

```java
static class Node<K,V> {
    final K key;
    volatile V val;
    volatile Node<K,V> next; // 同层后继

    // CAS 设置 val
    boolean casVal(V cmp, V val) { ... }
    // CAS 设置 next
    boolean casNext(Node<K,V> cmp, Node<K,V> val) { ... }
}

static class Index<K,V> {
    final Node<K,V> node;
    final Index<K,V> down;  // 下层索引
    volatile Index<K,V> right; // 右侧索引
}
```

### 线程安全机制

- **Node.val 和 Node.next 用 volatile**：保证可见性
- **CAS 更新**：casVal 和 casNext 原子操作
- **删除标记**：将 val 设为 null（逻辑删除），不影响正在查找的线程

### put 流程（简化）

```
put(key, value):
  1. 找到 key 应该插入的位置（从最高层往下找）
  2. CAS 创建新 Node
  3. 随机决定层数，创建 Index 层
  4. CAS 将 Index 插入各层链表
```

### vs ConcurrentHashMap

| 维度 | ConcurrentSkipListMap | ConcurrentHashMap |
|------|----------------------|-------------------|
| 有序性 | 按 key 自然排序 | 无序 |
| 查找复杂度 | O(log n) | O(1) 平均 |
| 内存 | 更多（多层索引） | 相对少 |
| 高并发性能 | 稳定（无扩容） | 扩容时有短暂降速 |
| 空间换时间 | 是 | 否 |

### NavigableMap API 详解

ConcurrentSkipListMap 实现完整的 NavigableMap 接口，提供丰富的有序查询方法：

```java
ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
map.put(10, "A"); map.put(20, "B"); map.put(30, "C");
map.put(40, "D"); map.put(50, "E");

// lower/ceil/higher —— 严格小于/小于等于/大于等于
map.lowerKey(30);      // 20（严格 < 30）
map.lowerEntry(30);    // 20=A
map.floorKey(30);      // 30（<= 30）
map.floorEntry(30);    // 30=C
map.ceilingKey(30);    // 30（>= 30）
map.ceilingEntry(30);  // 30=C
map.higherKey(30);     // 40（严格 > 30）
map.higherEntry(30);   // 40=D

// 边界操作
map.firstKey();         // 10（最小 key）
map.firstEntry();      // 10=A
map.lastKey();         // 50（最大 key）
map.lastEntry();       // 50=E

// 倒序视图
NavigableMap<Integer, String> desc = map.descendingMap();
desc.firstKey();       // 50（倒序最小 = 原最大）
desc.lastKey();        // 10（倒序最大 = 原最小）
```

| 方法 | 含义 | 示例（key=30 时） |
|------|------|------------------|
| `lowerKey(k)` | 严格 < k 的最大 key | 20 |
| `floorKey(k)` | ≤ k 的最大 key | 30 |
| `ceilingKey(k)` | ≥ k 的最小 key | 30 |
| `higherKey(k)` | 严格 > k 的最小 key | 40 |
| `firstKey()` | 最小 key | 10 |
| `lastKey()` | 最大 key | 50 |
| `descendingMap()` | 倒序视图 | — |

### 范围操作

```java
// subMap：左闭右开 [fromKey, toKey)
NavigableMap<Integer, String> sub = map.subMap(20, true, 40, false);
// keys: 20, 30  （20=true 包含，40=false 不包含）

// headMap：< toKey 或 ≤ toKey
map.headMap(30, true);   // keys <= 30: 10, 20, 30

// tailMap：≥ fromKey
map.tailMap(30, false);   // keys > 30: 40, 50

// descendingKeySet / navigableKeySet
NavigableSet<Integer> keys = map.navigableKeySet();     // 10,20,30,40,50
NavigableSet<Integer> rev  = map.descendingKeySet();   // 50,40,30,20,10
```

### 适用场景

- 需要**有序**的并发 Map（如排行榜、范围查询）
- 需要**范围操作**（subMap、headMap、tailMap）
- 高并发且不需要扩容（跳表天然支持动态增长）
- **Navigable API**：区间统计、排名、倒序遍历

## 易错点与踩坑

### 1. 跳表层数随机导致性能不稳定

```java
// ❌ 误以为 CSLM 的性能总是 O(log n)
// 实际上：跳表层数是概率分布，不是保证

// 随机层数生成（简化）：
// level 1 = 100%
// level 2 = 50%
// level 3 = 25%
// level 4 = 12.5%
// ...

// ⚠️ 最坏情况：所有节点的层数都是 1，退化为链表，O(n)
// 概率极低，但不是不可能

// ✅ 实际应用中，跳表的平均性能更稳定
// ✅ Redis 选择跳表而非红黑树，正是因为没有最坏情况
```

### 2. 删除操作是逻辑删除，不立即释放内存

```java
ConcurrentSkipListMap<String, Integer> map = new ConcurrentSkipListMap<>();
map.put("A", 1);
map.put("B", 2);
map.put("C", 3);

// ❌ 误以为 remove 后内存立即释放
map.remove("B");  // 逻辑删除：val = null，索引节点可能保留

// ⚠️ 问题：
// 1. 底层 Node.val 设为 null（逻辑删除）
// 2. 索引 Index 节点可能还没清理
// 3. 高并发删除时，索引可能堆积

// ✅ 解决方案：定期重建跳表（类似 HashMap 的 resize）
```

### 3. NavigableMap 子视图的弱一致性

```java
ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
map.put(1, "A"); map.put(5, "E"); map.put(10, "J");

// ❌ subMap 不是独立副本，是视图
NavigableMap<Integer, String> sub = map.subMap(2, true, 8, false);
// sub = {5: E}

// 在 sub 上删除元素
sub.remove(5);

// ⚠️ 原 map 也会被删除
System.out.println(map.containsKey(5));  // false！

// ✅ 如果需要独立副本，需要手动复制
Map<Integer, String> copy = new ConcurrentSkipListMap<>(sub);
```

## 关联知识点
