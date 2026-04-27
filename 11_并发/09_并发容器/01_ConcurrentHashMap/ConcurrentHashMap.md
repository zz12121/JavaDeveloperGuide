
# ConcurrentHashMap

## 核心结论

ConcurrentHashMap 是并发环境下最常用的 Map 实现。**JDK7 使用分段锁（Segment），JDK8 改为 CAS + synchronized + Node 数组**。JDK8 的设计在并发度、内存占用和性能上全面优于 JDK7。

## 深度解析

### JDK7 vs JDK8 对比

| 维度 | JDK7 | JDK8 |
|------|------|------|
| 数据结构 | Segment 数组 + HashEntry 数组 | Node 数组 + 链表 + 红黑树 |
| 锁粒度 | Segment 级别（默认 16 段） | 桶（bin）级别（首节点） |
| 锁实现 | ReentrantLock | CAS + synchronized |
| 并发度 | 默认 16（Segment 数量） | 数组长度（可到 2^30） |
| Hash 冲突 | 链表 | 链表 → 红黑树（阈值 8） |
| get 操作 | volatile 读 | volatile 读（无锁） |
| size 计算 | 各 Segment 的 size 累加 | baseCount + CounterCell |
| 扩容 | 每段独立扩容 | 多线程协助扩容（transfer） |

### JDK8 结构

```
ConcurrentHashMap
├── table: Node<K,V>[]（volatile）
│   ├── Node（链表节点）
│   ├── TreeNode（红黑树节点）
│   └── ForwardingNode（扩容标记，hash=-1）
├── baseCount: long（基础计数）
├── counterCells: CounterCell[]（分段计数数组）
├── sizeCtl: int（控制初始化和扩容）
├── transferIndex: int（扩容迁移索引）
└── nextTable: Node[]（扩容时的新数组）
```

### 关键设计

1. **CAS + synchronized 细粒度锁**：只锁当前桶的头节点，不锁整个 Segment
2. **volatile Node[]**：保证 get 操作的可见性
3. **ForwardingNode**：扩容时标记已迁移的桶（hash = -1）
4. **红黑树优化**：链表长度 ≥ 8 且数组长度 ≥ 64 时转红黑树
5. **多线程协助扩容**：其他线程可以参与数据迁移

### 基本使用

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// 线程安全操作
map.put("key", 1);
map.get("key");
map.remove("key");
map.putIfAbsent("key", 2);
map.computeIfAbsent("key", k -> 10);
map.merge("key", 1, Integer::sum);

// 原子操作
map.putIfAbsent("a", 1);       // 不存在才 put
map.replace("a", 1, 2);         // CAS 替换
map.getOrDefault("a", 0);       // 不存在返回默认值
```

# ConcurrentHashMap 1.7

## 核心结论

JDK7 的 ConcurrentHashMap 使用**分段锁（Segment）**：将数据分为 N 个 Segment，每个 Segment 是一个独立的 HashMap，继承 ReentrantLock，各自加锁互不影响。默认并发度 16。

## 深度解析

### 数据结构

```
ConcurrentHashMap
├── Segment[]（16个，继承 ReentrantLock）
│   ├── Segment[0]
│   │   └── HashEntry[]（桶数组）
│   │       ├── HashEntry → HashEntry → ...（链表）
│   ├── Segment[1]
│   │   └── HashEntry[]
│   └── ...
```

### Segment 源码

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    transient volatile HashEntry<K,V>[] table;
    transient int count;           // 元素数量（volatile）
    transient int modCount;        // 修改次数
    transient int threshold;       // 扩容阈值
    final float loadFactor;        // 负载因子
}

static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;              // volatile 保证可见性
    volatile HashEntry<K,V> next;  // volatile 保证链表可见性
}
```

### put 流程

```
put(key, value)
  1. hash(key) → 定位 Segment（二次哈希）
  2. segment.lock() → 获取该段的锁
  3. 检查是否需要扩容（threshold）
  4. 定位桶 → 遍历链表
     → key 已存在：更新 value
     → key 不存在：头插法创建新 HashEntry
  5. segment.unlock() → 释放锁
```

### get 流程

```
get(key)
  1. hash(key) → 定位 Segment
  2. 定位桶 → 遍历链表
  3. 找到返回 value，未找到返回 null
  4. 全程不加锁
  5. volatile 读保证 value 和 next 的可见性
```

### size 计算

```java
// 尝试计算所有 Segment 的 count 总和
// 如果计算过程中 count 变化（modCount 不一致），则重试
// 重试 2 次后，如果还变化，锁住所有 Segment 再计算
```

### 优缺点

**优点**：

- 锁粒度从整个 Map 缩小到 Segment，默认 16 段并发
- get 不加锁，性能高

**缺点**：

- 并发度固定为 Segment 数量，不能动态扩展
- Hash 冲突严重时链表长，查询效率低（无红黑树优化）
- Segment 本身存储 HashEntry 数组，内存开销大
- 扩容时只扩容单个 Segment，不够灵活

# ConcurrentHashMap 1.8

## 核心结论

JDK8 放弃分段锁，改用 **Node 数组 + CAS + synchronized + 红黑树**。锁粒度从 Segment 缩小到单个桶的首节点，并发度等于数组长度。引入 `sizeCtl` 控制初始化和扩容。

## 深度解析

### 数据结构

```
table: volatile Node<K,V>[]
  ├── Node（普通链表节点，hash≥0）
  ├── TreeNode（红黑树节点，hash≥0）
  ├── ForwardingNode（扩容标记，hash=-1）
  ├── ReservationNode（reserve时占位，hash=-3）
  └── TreeBin（红黑树根节点代理，hash=-2）
```

### 关键字段

```java
// Node 节点
static class Node<K,V> {
    final int hash;
    final K key;
    volatile V val;        // volatile
    volatile Node<K,V> next; // volatile
}

// 控制字段
private transient volatile int sizeCtl;
// < 0: 正在初始化（-1）或扩容（-(1+活跃线程数)）
// = 0: 未初始化，默认容量
// > 0: 下一次扩容的阈值（或初始化容量）
```

### Node 数组的 volatile 语义

```java
transient volatile Node<K,V>[] table;

// 读操作用 volatile 读（Unsafe.getObjectVolatile）
// 写操作用 CAS（Unsafe.compareAndSwapObject）
// 保证数组的引用可见性和节点更新的原子性
```

### 红黑树转换条件

```
链表 → 红黑树：
  链表长度 >= 8 且 数组长度 >= 64

红黑树 → 链表（退化）：
  扩容时拆分后节点数 <= 6 或 数组长度 < 64
```

### ForwardingNode

```java
// 扩容时，已迁移的桶放置 ForwardingNode
// hash = -1，find() 方法委托给 nextTable
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(-1, null, null, null);
        this.nextTable = tab;
    }
}
```

- 遇到 ForwardingNode 说明该桶已迁移到新数组
- 其他线程 put/get 时可以帮助完成扩容

### 树化过程

```
链表树化（treeifyBin）：
  1. 检查数组长度 < 64 → 优先扩容（不树化）
  2. 数组长度 >= 64 → 将链表转为 TreeBin
  3. TreeBin 包装 TreeNode，维护红黑树根节点
  4. TreeBin 用 synchronized 加锁（而非 TreeNode）
```

### sizeCtl 的状态变化

```
初始化：
  new CHM() → sizeCtl = 0
  new CHM(16) → sizeCtl = 16（指定的初始容量）
  initTable() → sizeCtl = -1（标记初始化中）→ sizeCtl = capacity * 0.75

扩容：
  sizeCtl > 0 且 size 达到阈值 → sizeCtl = -(1+1)（1个线程扩容）
  其他线程协助 → sizeCtl 递减
  扩容完成 → sizeCtl = newCapacity * 0.75
```

# ConcurrentHashMap put流程

## 核心结论

JDK8 的 put 流程：**hash 定位 → CAS 空槽 → synchronized 锁头节点 → 链表尾插/红黑树插入**。遇到 ForwardingNode 则帮助扩容。

## 深度解析

### 完整流程

```
putVal(key, value, onlyIfAbsent)
  1. 计算 hash（扰动函数，高低16位异或）
  2. 检查 table 是否为空 → initTable()（CAS 初始化）
  3. (n-1) & hash 定位桶索引 i
  4. table[i] == null ?
     → CAS 设置新 Node（无锁）
  5. table[i].hash == -1 ?
     → ForwardingNode，helpTransfer() 帮助扩容
  6. synchronized(table[i]) 锁头节点
     → hash >= 0（链表）→ 遍历链表尾插，存在则更新
     → hash == -2（TreeBin）→ putTreeVal 红黑树插入
  7. addCount() → 增加 size，检查是否需要扩容
```

### 源码（简化）

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());  // 扰动函数
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 1. 初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        tab = initTable();
    
    // 2. CAS 空槽
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = new Node<K,V>(hash, key, value, null); // CAS
    
    // 3. ForwardingNode → 帮助扩容
    else if (p.hash == -1)
        tab = helpTransfer(tab, p);
    
    // 4. 锁头节点，操作链表/红黑树
    else {
        V oldVal = null;
        synchronized (p) {
            if (tabAt(tab, i) == p) {
                // 链表
                if (p.hash >= 0) {
                    for (Node<K,V> e = p;;) {
                        K ek;
                        if (e.hash == hash &&
                            ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                            oldVal = e.val;
                            if (!onlyIfAbsent)
                                e.val = value;
                            break;
                        }
                        Node<K,V> pred = e;
                        if ((e = e.next) == null) {
                            pred.next = new Node<>(hash, key, value, null);
                            break;
                        }
                    }
                }
                // 红黑树
                else if (p instanceof TreeBin)
                    oldVal = ((TreeBin<K,V>)p).putTreeVal(hash, key, value);
            }
        }
        // 5. 增加计数
        addCount(1L, binCount);
    }
    return oldVal;
}
```

### 各阶段加锁策略

|阶段|锁机制|说明|
|---|---|---|
|初始化|CAS|只有一个线程能初始化成功|
|空槽插入|CAS|无锁，快速路径|
|帮助扩容|无锁（有同步）|多线程协作|
|链表/树操作|synchronized(头节点)|锁粒度极细|
|size 更新|CAS + LongAdder|分段计数|

### 关键细节

1. **扰动函数**：`spread(h) = (h ^ (h >>> 16)) & 0x7FFFFFFF`，减少碰撞
2. **尾插法**：JDK8 改为尾插法（JDK7 HashMap 用头插法导致死循环）
3. **volatile 读 tab[i]**：`tabAt(tab, i)` 用 `Unsafe.getObjectVolatile`
4. **CAS 写 tab[i]**：`casTabAt(tab, i, null, node)` 用 `Unsafe.compareAndSwapObject`

# ConcurrentHashMap get

## 核心结论

JDK8 的 ConcurrentHashMap get 操作**全程不加锁**，通过 volatile 读保证可见性。遇到 ForwardingNode 转发到新数组查找。

## 深度解析

### get 源码

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 1. 检查头节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 2. hash < 0（TreeBin / ForwardingNode）
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 3. 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

### 为什么不加锁？

1. **Node.val 用 volatile 修饰**：保证多线程下读到最新值
2. **Node.next 用 volatile 修饰**：保证遍历链表时能看到新节点
3. **table 用 volatile 修饰**：`tabAt()` 用 `Unsafe.getObjectVolatile` 读取

```java
static class Node<K,V> {
    final int hash;
    final K key;
    volatile V val;        // volatile 读保证可见性
    volatile Node<K,V> next; // volatile 遍历安全
}
```

### get 的可见性保证

```
线程A put：
  1. synchronized(f) 锁桶
  2. 创建新 Node 或更新 val
  3. 释放锁（happens-before）

线程B get：
  1. volatile 读 table[i]（tabAt）
  2. volatile 读 val / next
  3. 读到最新值
```

volatile 的 happens-before 语义保证了：put 释放锁之前的写操作对 get 操作可见。

### ForwardingNode 处理

```java
// hash < 0 时调用 find 方法
else if (eh < 0)
    return (p = e.find(h, key)) != null ? p.val : null;

// ForwardingNode.find() 转发到 nextTable
public Node<K,V> find(int h, Object k) {
    // 在 nextTable（新数组）中查找
    outer: for (Node<K,V>[] tab = nextTable;;) {
        // ... 在新数组中定位并查找
    }
}
```

### 弱一致性

ConcurrentHashMap 的 get 是**弱一致性**的：

- 不保证读到刚刚 put 的值
- 但保证不会读到 null（除了不存在的 key）
- 迭代器也是弱一致性的，不会抛 ConcurrentModificationException

# ConcurrentHashMap size

## 核心结论

JDK8 的 size 计算使用 **baseCount + CounterCell[]**，类似 LongAdder 的分段 CAS 机制。避免了 JDK7 中需要锁住所有 Segment 的高开销。

## 深度解析

### JDK7 方案

```java
// 尝试 2 次不加锁 sum
// 如果 modCount 变了，锁住所有 Segment 再算
int sum = 0;
for (Segment s : segments)
    sum += s.count; // 每次累加可能不一致
// 如果两次 sum 相同则返回，否则加锁重试
```

缺点：高并发下几乎总是需要锁住所有 Segment。

### JDK8 方案

```
size = baseCount + Σ CounterCell[i].value
```

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 : (n > Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i)
            sum += as[i].value;
    }
    return sum;
}
```

### addCount 流程

```java
// put 成功后调用
private void addCount(long x, int check) {
    CounterCell[] as; long b, v, s;
    // 1. CAS 更新 baseCount
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 2. CAS 失败 → 使用 CounterCell
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended); // 初始化或扩容 CounterCell
            return;
        }
        // ...
    }
}
```

### CounterCell 结构

```java
@jdk.internal.vm.annotation.Contended
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

- `@Contended` 避免伪共享（自动填充缓存行）
- 每个线程 hash 到不同的 CounterCell，减少 CAS 竞争

### 设计思想（与 LongAdder 一致）

```
LongAdder / ConcurrentHashMap.size
├── base: 一个基础值（低竞争时使用）
└── cells[]: 多个计数槽（高竞争时使用）
    ├── cells[0]: 线程A 的计数
    ├── cells[1]: 线程B 的计数
    ├── cells[2]: 线程C 的计数
    └── ...
    
size = base + Σ cells[i].value
```

# ConcurrentHashMap扩容

## 核心结论

JDK8 的扩容支持**多线程协助**：一个线程发起扩容后，其他 put/get 的线程可以参与数据迁移。通过 `transferIndex` 逆向分配迁移任务，`ForwardingNode` 标记已迁移的桶。

## 深度解析

### 扩容触发

```java
// addCount 中检查
if (check >= 0) {
    Node<K,V>[] tab, nt;
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
           (nt = tab.length) < MAXIMUM_CAPACITY) {
        // size >= sizeCtl（阈值）→ 触发扩容
        transfer(tab, null);
    }
}
```

### 多线程协助扩容

```
线程A put → addCount → 发现需要扩容 → transfer(tab, null)
线程B put → 发现 ForwardingNode → helpTransfer → transfer(tab, nextTable)
线程C get → 发现 ForwardingNode → 转发到 nextTable（不参与迁移）
```

### transfer 核心流程

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length;
    // 1. 创建新数组（容量翻倍）
    if (nextTab == null)
        nextTab = new Node<>(n << 1);
    
    // 2. 逆向分配迁移任务
    int stride = (NCPU > 1) ? (n >>> 3) / NCPU : n;
    // transferIndex 从 n 递减，每个线程负责 [transferIndex-stride, transferIndex) 的桶
    
    advance: for (;;) {
        // 分配当前线程负责的桶范围 [bound, i]
        while (i >= bound) {
            // 处理每个桶
            Node<K,V> f = tabAt(tab, i);
            if (f == null)          // 空桶 → 放置 ForwardingNode
                adv = casTabAt(tab, i, null, fwd);
            else if (f.hash == -1)  // 已迁移
                adv = true;
            else {
                synchronized (f) {  // 锁桶头节点
                    if (tabAt(tab, i) == f) {
                        // 链表迁移：拆分为低位和高位
                        // 红黑树迁移：拆分或退化为链表
                        // 放置 ForwardingNode 标记
                    }
                }
            }
        }
    }
}
```

### 链表拆分（核心算法）

```
原数组索引 i 的链表，迁移到新数组：
  新数组容量 = 2n
  新索引 = hash & (2n - 1)

拆分规则：
  如果 (hash & n) == 0 → 新索引 i（低位）
  如果 (hash & n) != 0 → 新索引 i + n（高位）

  n = 原数组长度，也是扩容位
```

```
原数组 table[3]:
  节点A (hash & 8 == 0) → 新数组 nextTab[3]
  节点B (hash & 8 != 0) → 新数组 nextTab[3+8=11]
  节点C (hash & 8 == 0) → 新数组 nextTab[3]
```

### sizeCtl 与协助线程

```java
// 扩容线程管理
sizeCtl 初始 = -(1 + 1) = -2（1个线程）
新线程协助 → CAS sizeCtl - 1
线程完成 → CAS sizeCtl + 1
最后一个完成 → sizeCtl = newCapacity * 0.75
```

### ForwardingNode 协助

```java
// put/get 遇到 ForwardingNode
if (tabAt(tab, i).hash == -1) {
    // put: helpTransfer 参与扩容
    // get: 在 nextTable 中查找
}
```

# ConcurrentHashMap死循环

## 核心结论

JDK7 的 HashMap 在**多线程并发扩容**时，链表使用头插法可能导致**环形链表**，引发死循环。ConcurrentHashMap 1.7 通过分段锁避免了这个问题。JDK8 改用尾插法，从根本上解决了环形链表问题。

## 深度解析

### JDK7 HashMap 头插法导致环形链表

```
扩容前 table[3]:  A → B → null
扩容后: A 和 B 都 rehash 到 table[11]

线程1：newTable[11] = B → A → null（正常）
线程2：在某个时刻被挂起，看到的还是旧链表 A → B → null

线程2 恢复后：
  e = A, next = B
  头插：newTable[11] = A → null
  e = B, next = null（但此时 A.next 可能已被线程1改为 B）
  头插：newTable[11] = B → A → B → A → ...（环形链表！）
```

```
正常情况：头插法
  A → B → null

多线程头插法：
  A → B → A → B → ...（死循环）
```

### 为什么 ConcurrentHashMap 1.7 没有这个问题

- **Segment 加锁**：每个 Segment 扩容时独占锁，不允许其他线程操作该 Segment
- 不同 Segment 可以并发扩容，但同一 Segment 内不会出现并发扩容

### JDK8 如何解决

1. **尾插法**：链表插入在尾部，不会反转链表顺序
2. **ConcurrentHashMap**：用 synchronized 锁桶头节点，同一桶不会并发扩容
3. **ForwardingNode**：标记已迁移的桶，防止重复处理

```
JDK8 尾插法：
  扩容前: A → B → null
  扩容后: A → B → null（顺序不变）

  即使多线程参与迁移，每个桶也有 synchronized 保护
  不会出现环形链表
```

### 总结

|版本|数据结构|插入方式|并发扩容|环形链表|
|---|---|---|---|---|
|JDK7 HashMap|数组+链表|头插法|不支持|❌ 可能|
|JDK7 CHM|Segment+HashEntry|头插法|Segment锁|✅ 不会|
|JDK8 HashMap|数组+链表+红黑树|尾插法|不支持|✅ 不会|
|JDK8 CHM|Node数组+链表+红黑树|尾插法|synchronized桶锁|✅ 不会|
## 易错点与踩坑

### 1. computeIfAbsent 的死锁/性能陷阱（JDK8特有，JDK11+已修复）

```java
// ❌ JDK8 中的经典陷阱：递归死锁
ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();

// 线程A
Object value = cache.computeIfAbsent("key", k -> {
    // 这里需要查询同样的 key
    return cache.get("key");  // 可能触发同一线程的 computeIfAbsent 嵌套调用
});

// 实际场景：缓存预热+懒加载组合
public Object getOrLoad(String key) {
    return cache.computeIfAbsent(key, k -> {
        Object data = loadFromDB(k);
        // 业务逻辑可能触发其他 computeIfAbsent
        return data;
    });
}
```

**问题原因**：JDK8 中 computeIfAbsent 会检查当前节点是否正在被其他线程创建，如果是则自旋等待。如果递归调用同一个 key，可能导致栈溢出或死锁。

**JDK11+修复**：不再自旋等待，改为直接返回 null 或创建新值。

### 2. size() vs mappingCount() 的坑

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ size() 返回 int，可能溢出
// 如果 map 超过 Integer.MAX_VALUE (2^31-1)，会截断
int s = map.size();  // 可能不准确

// ✅ JDK8+ 推荐使用 mappingCount()
long count = map.mappingCount();  // 返回 long，不会溢出
```

**场景**：大数据量统计（如 UV 计数）时必须用 `mappingCount()`。

### 3. 迭代器弱一致性导致的数据丢失感

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("A", 1);
map.put("B", 2);

// ❌ 错误认知：迭代器应该看到实时数据
Iterator<Map.Entry<String, Integer>> iter = map.entrySet().iterator();

// 线程B同时修改
map.put("C", 3);
map.remove("A");

// 迭代器可能看到：跳过 C，部分元素可能重复或跳过
while (iter.hasNext()) {
    Map.Entry<String, Integer> entry = iter.next();
    System.out.println(entry.getKey());
}
```

**原理**：CHM 迭代器遍历的是桶的快照，不会抛 ConcurrentModificationException，但也不保证遍历所有元素。

### 4. 红黑树退化时节点丢失

```java
// 扩容过程中，如果数组长度 < 64，会优先扩容而非树化
// 扩容拆分后，红黑树可能退化为链表

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// 连续 put 触发扩容：数组从 16 → 32 → 64
// 在扩容期间，某桶的树可能被打散

// ⚠️ 扩容后再树化：需要再次达到阈值（8个节点）
// 中间状态可能导致查找效率下降
```

**建议**：避免频繁的 put+remove 交替操作，这会导致扩容和树化退化循环。

### 5. key/value 都不允许为 null（与 HashMap 不同）

```java
// ❌ HashMap 允许 null key/value
HashMap<String, Integer> hm = new HashMap<>();
hm.put(null, 1);    // ✅ OK
hm.put("a", null);  // ✅ OK

// ❌ ConcurrentHashMap 不允许 null
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.put(null, 1);   // NullPointerException
chm.put("a", null); // NullPointerException

// ❌ get() 返回 null 可能是两种情况：
// 1. key 不存在
// 2. key 存在但 value 是 null（不允许！）
// 无法区分！因此 CHM 禁止 null
```

**设计原因**：在并发环境下，无法区分"key不存在"和"key存在但值为null"，这会导致业务逻辑错误。

## 关联知识点

