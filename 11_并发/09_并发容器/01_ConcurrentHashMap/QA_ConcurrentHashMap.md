---
title: ConcurrentHashMap
tags:
  - Java/并发
  - 问答
  - 对比型
module: 09_并发容器
created: 2026-04-18
---

# ConcurrentHashMap

## Q1：ConcurrentHashMap JDK7 和 JDK8 有什么区别？

**A**：

| 维度 | JDK7 | JDK8 |
|------|------|------|
| 数据结构 | Segment 数组 + HashEntry | Node 数组 + 链表 + 红黑树 |
| 锁粒度 | Segment 级别（默认16段） | 桶级别（首节点） |
| 锁实现 | ReentrantLock | CAS + synchronized |
| 并发度 | 16（Segment数量） | 数组长度 |
| Hash冲突 | 链表 | 链表→红黑树 |

JDK8 用 CAS + synchronized 替代了 ReentrantLock，锁粒度更细，并发度更高，内存占用更低。

---

## Q2：为什么 JDK8 用 synchronized 替代 ReentrantLock？

**A**：

1. **synchronized 在 JDK6 后大幅优化**：偏向锁、轻量级锁，低竞争时性能不输 ReentrantLock
2. **内存占用更少**：每个 Node 不需要额外存储锁对象（JDK7 每个 HashEntry 需要）
3. **JVM 层面优化空间大**：synchronized 的锁信息存在对象头，JVM 可以做逃逸分析、锁消除
4. **大量数据统计表明**：大部分情况下桶竞争不激烈，synchronized 足够

---

## Q3：ConcurrentHashMap 有哪些原子操作方法？

**A**：

| 方法 | 说明 |
|------|------|
| `putIfAbsent(key, value)` | key 不存在时才 put |
| `replace(key, oldValue, newValue)` | CAS 替换 |
| `replace(key, newValue)` | key 存在时替换 value |
| `remove(key, value)` | key-value 都匹配才删除 |
| `computeIfAbsent(key, fn)` | key 不存在时计算并放入 |
| `computeIfPresent(key, fn)` | key 存在时重新计算 |
| `compute(key, fn)` | 始终计算 |
| `merge(key, value, fn)` | 合并操作 |

这些方法保证了操作的原子性，无需外部加锁。

---
```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

// 线程安全的 put/get
map.put("key", "value");
String v = map.get("key");  // 无锁读（volatile + CAS）

// 原子操作（不需要额外加锁）
map.putIfAbsent("key", "newValue");      // key 不存在才 put
map.replace("key", "old", "new");        // CAS 替换
map.computeIfAbsent("key", k -> "new");   // key 不存在时计算
map.merge("key", "update", (old, val) -> old + val);  // 合并

// JDK8 size：sumCount() 近似值（不加锁）
int size = map.mappingCount();
```


# ConcurrentHashMap 1.7

## Q1：JDK7 ConcurrentHashMap 的分段锁是怎么实现的？

**A**：

1. **数据分为多个 Segment**（默认 16 个），每个 Segment 继承 `ReentrantLock`
2. 每个 Segment 内部是一个独立的 `HashEntry[]` 数组 + 链表
3. put 时先 hash 定位 Segment，获取该 Segment 的锁，然后操作
4. 不同 Segment 的操作互不影响，实现并发

```
put 流程：
hash(key) → 定位 Segment → segment.lock() → 操作 HashEntry → segment.unlock()
```

---

## Q2：JDK7 的 ConcurrentHashMap get 为什么不需要加锁？

**A**：

1. `HashEntry.value` 和 `HashEntry.next` 都用 `volatile` 修饰
2. volatile 保证了多线程下的可见性
3. get 只读操作，读到的链表节点虽然可能是"旧的"，但不会读到 null 或错误数据
4. 注意：get 到的不一定是最新写入的值（弱一致性），但不会抛异常

---

## Q3：JDK7 分段锁有什么缺点？

**A**：

1. **并发度受限**：默认 16 个 Segment，最多 16 个线程并发写
2. **Hash 冲突无优化**：只有链表，冲突严重时退化为 O(n)
3. **内存开销大**：Segment 对象 + 每个 Segment 的 HashEntry 数组
4. **扩容不灵活**：只能单个 Segment 独立扩容
5. **size 计算复杂**：需要重试或锁住所有 Segment

# ConcurrentHashMap 1.8

## Q1：JDK8 ConcurrentHashMap 的锁机制是什么？

**A**：JDK8 使用 **CAS + synchronized** 组合：

- **CAS 操作**：初始化数组、空桶插入（头节点）、扩容相关操作
- **synchronized**：锁定桶的头节点（链表头或 TreeBin），粒度极细
- 对比 JDK7 的 ReentrantLock（Segment 级别），锁粒度从 Segment 缩小到单个桶

```java
// put 时锁定头节点
synchronized (f) {  // f 是桶的头节点
    // 操作链表或红黑树
}
```

---

## Q2：ForwardingNode 的作用是什么？

**A**：ForwardingNode 是扩容过程中的**占位标记**：

- hash = -1，放在已迁移完成的桶位置
- 其他线程操作时发现 ForwardingNode，知道正在扩容
- get 操作会转发到 `nextTable` 查找
- put 操作会帮助完成扩容

---

## Q3：sizeCtl 有哪些含义？

**A**：

|值|含义|
|---|---|
|0|未初始化|
|正数|初始容量 或 扩容阈值|
|-1|正在初始化|
|-(1+n)|正在扩容，n 为协助扩容的线程数|

sizeCtl 是 ConcurrentHashMap 的核心控制字段，用 CAS 更新，保证多线程下的状态一致性。

---

## Q4：链表什么时候转红黑树？什么时候退化？

**A**：

**树化条件**：链表长度 ≥ 8 **且** 数组长度 ≥ 64

- 如果数组 < 64，优先扩容而非树化

**退化条件**（untreeify）：

- 扩容拆分时节点数 ≤ 6
- 数组长度 < 64（扩容时自动退化）

# ConcurrentHashMap put流程

## Q1：描述 JDK8 ConcurrentHashMap 的 put 流程。

**A**：

```
put(key, value)
  1. spread(hash) 计算扰动后的 hash
  2. table 为空 → initTable() CAS 初始化
  3. (n-1) & hash 定位桶
  4. 桶为空 → CAS 插入新 Node（无锁快速路径）
  5. 桶头节点是 ForwardingNode → helpTransfer 帮助扩容
  6. synchronized(头节点) 锁桶
     → 链表：遍历，找到则更新，找不到则尾插
     → 红黑树：putTreeVal
  7. addCount 增加计数，检查是否扩容
```

---

## Q2：put 操作的加锁策略是什么？

**A**：分阶段使用不同策略：

1. **CAS 空槽**：桶为空时直接 CAS，无锁
2. **synchronized 头节点**：桶不为空时，只锁当前桶的头节点
3. **volatile 读**：`tabAt()` 读取桶节点，保证可见性

这种设计让大部分写操作不需要竞争同一把锁，不同桶可以并发写入。

---

## Q3：put 时遇到 ForwardingNode 怎么办？

**A**：ForwardingNode（hash=-1）表示该桶已迁移到新数组。

- put 时遇到 ForwardingNode → 调用 `helpTransfer()` 参与扩容
- 扩容完成后在新数组上重新执行 put
- get 时遇到 ForwardingNode → 转发到 `nextTable` 查找

这种"帮忙"机制让多个线程可以并行扩容，提高效率。

---

## Q4：ConcurrentHashMap 的 put 方法是线程安全的吗？复合操作呢？

**A**：

- **单次 put**：完全线程安全
- **复合操作不安全**：如"先 get 再 put"（check-then-act），两个操作之间可能被其他线程修改

复合操作需要使用原子方法：

```java
// 不安全
if (!map.containsKey(key)) {
    map.put(key, value); // 两步之间可能被其他线程插入
}

// 安全
map.putIfAbsent(key, value);    // 原子操作
map.computeIfAbsent(key, k -> value); // 原子操作
```

# ConcurrentHashMap get

## Q1：ConcurrentHashMap 的 get 为什么不需要加锁？

**A**：

1. `Node.val` 和 `Node.next` 都用 **volatile** 修饰，保证可见性
2. `table` 数组引用是 volatile 的，`tabAt()` 使用 `Unsafe.getObjectVolatile`
3. volatile 读保证了 happens-before 关系：put 中 synchronized 释放锁前的写操作，对 get 中的 volatile 读可见

所以 get 不加锁也能读到正确数据（弱一致性）。

---

## Q2：get 操作是强一致还是弱一致？

**A**：**弱一致性**。

- 不保证读到另一个线程刚刚 put 的值
- 但不会读到 null 或错误值（对于已存在的 key）
- 迭代器也是弱一致的，不会抛 `ConcurrentModificationException`

这是因为 get 不加锁，可能读到"稍旧"的值。对于大多数缓存场景足够。

---

## Q3：get 遇到 ForwardingNode 怎么处理？

**A**：

- `tabAt()` 读到 ForwardingNode（hash=-1），说明该桶正在扩容
- 调用 `ForwardingNode.find()` 转发到 `nextTable`（新数组）查找
- 在新数组中重新定位桶并查找 key
- 扩容完成后自动失效，后续 get 直接在新数组上查找

---

```java
// ConcurrentHashMap.get() 无锁读原理
// 1. 计算 hash → 定位桶
// 2. tabAt(i) 用 Unsafe.getObjectVolatile 读（volatile 读）
// 3. 首节点 hash == 目标 hash → 比较 key → 找到返回
// 4. hash < 0 → ForwardingNode → 转发到新数组查找
// 5. hash > 0 → 遍历链表/红黑树

ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("a", "1");

// get 无锁，弱一致性
String val = map.get("a");  // 可能读到旧值，但不会读到 null 或错误值
```

# ConcurrentHashMap size

## Q1：JDK8 ConcurrentHashMap 的 size 是怎么计算的？

**A**：使用 **baseCount + CounterCell[]** 方案：

```
size = baseCount + Σ CounterCell[i].value
```

- 低竞争时：CAS 直接更新 baseCount
- CAS 失败（多线程竞争）：创建 CounterCell 数组，每个线程 hash 到不同的 Cell 更新
- 最终求和：baseCount + 所有 CounterCell 的 value

这与 LongAdder 的设计思想一致。

---

## Q2：为什么不用一个 volatile 变量记录 size？

**A**：高并发下所有线程竞争一个 volatile 变量会导致严重的 CAS 失败，性能下降。

分段计数的优势：

1. 不同线程更新不同的 CounterCell，减少竞争
2. 读 size 时累加所有 Cell，虽然不精确但足够好
3. `@Contended` 注解避免伪共享

---

## Q3：size() 返回的是精确值吗？

**A**：**不是精确值**，是近似值。

- `sumCount()` 遍历 baseCount 和 CounterCell 数组求和
- 求和过程中可能有新的 put/remove 操作
- 返回的是某个时刻的"快照"，不保证精确

但对于绝大多数场景，这个近似值已经足够。

# ConcurrentHashMap扩容

## Q1：JDK8 ConcurrentHashMap 的多线程协助扩容是怎么实现的？

**A**：

1. **发起扩容**：addCount 检测 size >= sizeCtl，CAS 将 sizeCtl 设为负值，创建新数组（容量翻倍）
2. **任务分配**：通过 `transferIndex` 逆向分配迁移范围（从后往前），每个线程负责一段桶的迁移
3. **协助扩容**：其他线程 put/get 时发现 ForwardingNode，调用 `helpTransfer` 参与迁移
4. **线程计数**：sizeCtl 用负值记录参与扩容的线程数 `-(1+n)`
5. **完成标志**：最后一个完成的线程将 sizeCtl 设为新阈值

---

## Q2：扩容时链表是怎么拆分的？

**A**：利用容量翻倍的特点，**一个桶拆分为两个**：

```
原数组 table[i] 的链表：
  hash & n == 0 → 新数组 nextTab[i]（低位）
  hash & n != 0 → 新数组 nextTab[i+n]（高位）

n = 原数组长度
```

每个节点只需要检查 hash 的第 n 位（扩容位），决定分配到低位还是高位。不需要重新计算 hash。

---

## Q3：ForwardingNode 在扩容中起什么作用？

**A**：

1. **标记已迁移**：已迁移的桶放置 ForwardingNode（hash=-1）
2. **触发协助**：put 操作遇到 ForwardingNode 时调用 helpTransfer 参与
3. **转发读操作**：get 操作遇到 ForwardingNode 转发到 nextTable
4. **防止重复迁移**：多个线程看到同一个桶时不会重复处理


# ConcurrentHashMap死循环

## Q1：JDK7 HashMap 为什么多线程扩容会死循环？

**A**：因为 JDK7 HashMap 使用**头插法**迁移链表：

1. 线程 A 和线程 B 同时扩容
2. 头插法会反转链表顺序
3. 线程 B 读到线程 A 已反转的部分链表
4. 线程 B 继续头插时，将节点插入到已反转的链表中
5. 形成 `A → B → A → B → ...` 环形链表
6. 后续 get 操作在环上无限遍历 → CPU 100%，死循环

---

## Q2：ConcurrentHashMap 1.7 有死循环问题吗？

**A**：**没有**。

JDK7 ConcurrentHashMap 使用分段锁（Segment extends ReentrantLock）：

- 每个 Segment 扩容时独占锁
- 同一 Segment 内不会并发扩容
- 不同 Segment 的扩容互不影响

所以虽然 HashEntry 也用头插法，但分段锁保证了不会有并发扩容问题。

---

## Q3：JDK8 如何解决环形链表问题？

**A**：

1. **尾插法**：迁移时不反转链表顺序，消除了环形链表的根本原因
2. **synchronized 桶锁**：迁移时锁桶头节点，同一桶不会并发操作
3. **ForwardingNode 标记**：已迁移的桶用 ForwardingNode 标记，防止重复处理

即使多线程协助扩容，也不会产生环形链表。

---

```java
// JDK7 HashMap 头插法导致环形链表示意
// 线程A 扩容：链表 [3→7→5] → 头插法反转 [5→7→3]
// 线程B 同时扩容，读到部分反转的链表
// 结果：[3→5→3] 或 [5→3→5] 环形链表
// get(5) → 死循环，CPU 100%

// JDK8 解决方案：尾插法，保持链表顺序
// transfer() 中：
// Node<K,V> loHead, loTail;  // 低位链表头尾
// while (p != null) {
//     Node<K,V> next = p.next;
//     if ((p.hash & oldCap) == 0) {
//         if (loTail == null) loHead = p;
//         else loTail.next = p;  // 尾插，不反转
//         loTail = p;
//     }
//     p = next;
// }
```