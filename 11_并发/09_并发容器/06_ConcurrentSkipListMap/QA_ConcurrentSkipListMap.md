---
title: ConcurrentSkipListMap
tags:
  - Java/并发
  - 问答
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentSkipListMap

## Q1：跳表是什么？为什么适合并发场景？

**A**：跳表是多层有序链表，通过概率随机生成上层索引实现快速查找：

- **查找 O(log n)**：从最高层开始，逐层下降，类似二分查找
- **插入/删除 O(log n)**：随机决定新节点的层数
- **天然有序**：key 始终有序排列
- **适合并发**：通过 CAS 更新节点指针，不需要全局锁；删除时逻辑标记（val=null），不影响正在查找的线程

对比红黑树：跳表实现更简单，并发控制更容易（红黑树的旋转操作在并发下很复杂）。

---

## Q2：ConcurrentSkipListMap 和 ConcurrentHashMap 怎么选？

**A**：

- **需要有序/范围查询**：选 ConcurrentSkipListMap（支持 subMap、headMap 等）
- **不需要有序，追求最高性能**：选 ConcurrentHashMap（O(1) 查找）
- **排行榜、时序数据**：选 ConcurrentSkipListMap
- **缓存、计数器**：选 ConcurrentHashMap

---

## Q3：ConcurrentSkipListMap 的删除是怎么实现的？

**A**：**逻辑删除 + 物理清理**：

1. **逻辑删除**：CAS 将节点的 val 设为 null
2. 其他查找线程遇到 val=null 的节点会跳过
3. **物理清理**：后续操作时，将 val=null 的节点从链表中移除

这种两阶段删除避免了在删除过程中阻塞查找操作。



> **代码示例：ConcurrentSkipListMap 基本操作与范围查询**

```java
ConcurrentSkipListMap<String, Integer> leaderboard = new ConcurrentSkipListMap<>();

// 并发安全的有序插入
leaderboard.put("Alice", 95);
leaderboard.put("Bob", 87);
leaderboard.put("Charlie", 92);

// 范围查询（ConcurrentHashMap 不支持）
Map<String, Integer> top = leaderboard.subMap("B", true, "D", true);
// 结果: {Bob=87, Charlie=92}

// 排行榜场景：获取前 N 名
leaderboard.forEach((k, v) -> System.out.println(k + ": " + v));
// 输出按 key 有序: Alice: 95, Bob: 87, Charlie: 92
```

---

## Q4：NavigableMap 的 lower/floor/ceiling/higher 有什么区别？

**A**：严格性是关键区分：

| 方法 | 严格性 | 语义 | 示例（key=30 时） |
|------|--------|------|------------------|
| `lowerKey(k)` | 严格 < | 小于 k 的最大 key | 20 |
| `floorKey(k)` | ≤ | 小于等于 k 的最大 key | 30 |
| `ceilingKey(k)` | ≥ | 大于等于 k 的最小 key | 30 |
| `higherKey(k)` | 严格 > | 大于 k 的最小 key | 40 |

```java
map.put(10,"A"); map.put(20,"B"); map.put(30,"C"); map.put(40,"D");

map.lowerKey(30);    // 20（< 30）
map.floorKey(30);    // 30（<= 30）
map.ceilingKey(30);  // 30（>= 30）
map.higherKey(30);   // 40（> 30）
```

记忆技巧：**lower = 下面，floor = 地板（到底），ceiling = 天花板（到顶），higher = 上面**。

---

## Q5：ConcurrentSkipListMap 的 subMap 是强一致还是弱一致？

**A**：**弱一致性**。subMap 返回的视图依赖原 Map 的底层跳表结构，对原 Map 的并发修改可能影响 subMap：

```java
ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
map.put(10, "A"); map.put(20, "B"); map.put(30, "C");

NavigableMap<Integer, String> sub = map.subMap(10, true, 30, false);
// sub: {10=A, 20=B}（30 不包含）

// 在 sub 视图上的操作：
sub.put(25, "X");   // 写入原 map（25 落在子视图范围内）
// map 现在: {10=A, 20=B, 25=X, 30=C}

map.remove(20);     // 原 map 删除，sub 视图也受影响
// sub 现在: {10=A, 25=X}（20 已移除）
```

**注意事项**：
- subMap 上的写入会直接影响原 Map
- 原 Map 的并发修改可能导致 subMap 的迭代器行为不确定
- 如果需要完全隔离的快照，需要手动复制

---

## Q6：ConcurrentSkipListMap 为什么比 TreeMap（加锁）更适合并发？

**A**：核心原因是**无锁设计**和**两阶段删除**：

| 维度 | TreeMap + synchronized | ConcurrentSkipListMap |
|------|----------------------|----------------------|
| 并发控制 | synchronized 全局锁 | CAS + volatile（无锁） |
| 读操作 | 需获取锁 | 无锁，读写可并发 |
| 删除 | 物理删除（阻塞查找） | val=null 逻辑删除 + 后台清理 |
| 锁粒度 | 全局 | 节点级别 |
| 复杂度 | 简单 | 跳表 + CAS |

```java
// TreeMap：所有操作串行
synchronized (treeMap) {
    treeMap.put(k, v);    // 写入阻塞所有读写
    treeMap.get(k);       // 读取也需获取锁
}

// ConcurrentSkipListMap：读写无锁并发
map.put(k, v);            // CAS，无锁
map.get(k);               // volatile 读，无锁
```

跳表的优势：不需要旋转操作（vs 红黑树的左旋/右旋），CAS 更新指针即可实现并发安全。

