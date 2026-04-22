---
title: HashMap 底层实现
tags:
  - Java/集合框架
  - 源码型
module: 05_集合框架
created: 2026-04-18
---

# HashMap 底层实现（数组 + 链表 + 红黑树）

## 数据结构

JDK 8 中 HashMap 采用 **数组 + 链表 + 红黑树** 结构：

```
table[]（数组，也叫"桶数组"）
  ├── [0] → null
  ├── [1] → Node → Node → Node  （链表）
  ├── [2] → TreeNode(红黑树)
  ├── [3] → Node
  ├── [4] → null
  └── ...
```

### 关键字段

```java
public class HashMap<K, V> extends AbstractMap<K, V> {
    transient Node<K, V>[] table;     // 桶数组
    transient int size;                // 键值对数量
    int threshold;                     // 扩容阈值 = capacity * loadFactor
    final float loadFactor;            // 负载因子，默认 0.75

    static final int DEFAULT_INITIAL_CAPACITY = 16;  // 默认初始容量
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;   // 链表转红黑树阈值
    static final int UNTREEIFY_THRESHOLD = 6; // 红黑树退化回链表阈值
    static final int MIN_TREEIFY_CAPACITY = 64; // 桶数组至少 64 才树化
}
```

### Node 结构

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;     // 哈希值
    final K key;        // 键
    V value;            // 值
    Node<K, V> next;    // 下一个节点（链表指针）

    Node(int hash, K key, V value, Node<K, V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

## JDK 7 vs JDK 8 对比

| 维度 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | 数组 + 链表 | 数组 + 链表 + **红黑树** |
| 链表插入方式 | **头插法** | **尾插法** |
| 扩容时链表 | 可能产生**环形链表**（多线程下） | 尾插法避免 |
| 哈希计算 | 多次扰动 | 简化扰动（1 次） |
| 默认初始容量 | 16 | 16（懒初始化） |

## 哈希计算

```java
static final int hash(Object key) {
    int h;
    // 高 16 位与低 16 位异或，减少碰撞
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 定位桶索引
int index = (n - 1) & hash;  // n 是数组长度，必须是 2 的幂次
```

**为什么要异或高 16 位？**
- 数组长度一般较小（< 2^16），`hash & (n-1)` 只用到低位
- 将高位特征混入低位，减少哈希碰撞

**为什么数组长度必须是 2 的幂次？**
- `hash % n` 等价于 `hash & (n - 1)`，位运算比取模更快
- 且 `n - 1` 的二进制全为 1，能充分利用 hash 的低位信息

## 链表 → 红黑树

| 条件 | 动作 |
|------|------|
| 链表长度 ≥ 8 **且** 数组长度 ≥ 64 | 链表 → **红黑树**（treeifyBin） |
| 红黑树节点数 ≤ 6 | 红黑树 → **链表**（untreeify） |
| 链表长度 ≥ 8 **但** 数组长度 < 64 | **扩容**而非树化 |

## 核心操作时间复杂度

| 操作 | 平均 | 最坏 |
|------|------|------|
| put | O(1) | O(log n)（树化后） |
| get | O(1) | O(log n)（树化后） |
| remove | O(1) | O(log n)（树化后） |

# HashMap 扩容机制（负载因子 0.75）

## 扩容触发条件

当 `size > threshold`（即 `size > capacity × loadFactor`）时触发扩容。

```
threshold = capacity × loadFactor
默认：threshold = 16 × 0.75 = 12
即第 13 个元素插入时触发扩容
```

## 负载因子为什么是 0.75

|负载因子|效果|
|---|---|
|太大（如 1.0）|空间利用率高，但哈希碰撞多，链表长，查询慢|
|太小（如 0.5）|碰撞少，查询快，但空间浪费严重|
|**0.75（默认）**|时间和空间的**平衡点**，泊松分布计算得出|

根据泊松分布，当负载因子为 0.75 时，链表长度达到 8 的概率仅为 **0.00000006**，极其罕见，所以红黑树是一个"保底"机制。

## 扩容过程

### JDK 8 扩容

```
1. 创建新数组，容量 = 旧容量 × 2
2. 遍历旧数组每个桶
3. 对每个节点重新计算桶位置
4. 通过 hash & oldCapacity 判断新位置：
   - 结果为 0 → 位置不变
   - 结果为 1 → 位置 = 旧位置 + 旧容量
5. 将节点分配到新数组的两个链表（loHead/hiHead）
```

### 关键优化：JDK 8 不需要重新计算 hash

```java
if ((e.hash & oldCap) == 0)
    // 新位置 = 旧位置（低位链表）
else
    // 新位置 = 旧位置 + oldCapacity（高位链表）
```

因为 `oldCap` 是 2 的幂次，只有 hash 在 `oldCap` 对应的那个 bit 位不同。只需要看那一位是 0 还是 1 就能确定新位置，**不需要重新做 `hash & (newCap - 1)`**。

### 示例

```
旧容量 = 16，旧掩码 = 15 (0b1111)
新容量 = 32，新掩码 = 31 (0b11111)

hash = 0b...10010
  旧位置 = hash & 15  = 0b0010 = 2
  新位置 = hash & 31  = 0b10010 = 18

判断：hash & 16 = 0b10000 ≠ 0 → 高位链表
  新位置 = 旧位置 + 16 = 2 + 16 = 18 ✅
```

## JDK 7 vs JDK 8 扩容对比

|维度|JDK 7|JDK 8|
|---|---|---|
|扩容后链表重组|**头插法**（逆序）|**尾插法**（保持顺序）|
|多线程死循环|⚠️ 可能发生（头插法导致环形链表）|✅ 不会（尾插法）|
|元素迁移|全部重新 hash|利用高低位拆分，避免重算|

## JDK 7 并发扩容死循环

```
线程A 扩容，头插法迁移元素到新数组
线程B 也扩容，也在迁移
由于头插法逆序 + 并发读写，可能形成：
  node.next 指向已迁移到新数组的节点
  形成环形链表 → get() 时死循环
```

JDK 8 改为尾插法，彻底解决了这个问题。但 HashMap 仍然是**非线程安全**的，多线程应使用 ConcurrentHashMap。

# HashMap put 流程

## 完整流程图

```
put(key, value)
│
├── 1. 计算 hash = hash(key)
│       h = key.hashCode()
│       hash = h ^ (h >>> 16)    // 高低位异或
│
├── 2. 定位桶 index = (n - 1) & hash
│
├── 3. 桶为空？
│   ├── 是 → 直接新建 Node 放入（tab[i] = newNode）
│   │       → size++ → 判断是否扩容 → 结束
│   │
│   └── 否 → 遍历桶中的链表/红黑树
│       │
│       ├── 4. 找到相同 key？（hash 相同 && (key == k || key.equals(k))）
│       │   ├── 是 → 替换旧 value，返回旧 value
│       │   │
│       │   └── 否 → 继续遍历
│       │
│       ├── 5. 桶中是红黑树？
│       │   └── 是 → 调用红黑树插入 putTreeVal()
│       │
│       └── 6. 链表尾部插入
│           ├── binCount >= 7 且数组长度 >= 64？
│           │   └── 是 → treeifyBin() 链表转红黑树
│           └── 否 → 继续链表
│
├── 7. size++ 
│   └── size > threshold？ → resize() 扩容
│
└── 结束
```

## 核心源码（简化版）

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K, V>[] tab; Node<K, V> p; int n, i;

    // 1. 数组为空则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 2. 桶为空，直接放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K, V> e; K k;

        // 3. 桶头节点就是目标 key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // 4. 红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);

        // 5. 链表遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null); // 尾插
                    if (binCount >= TREEIFY_THRESHOLD - 1) // >= 7
                        treeifyBin(tab, hash);  // 尝试树化
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;  // 找到相同 key
                p = e;
            }
        }

        // 6. key 已存在，替换 value
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }

    ++modCount;
    // 7. 判断扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## 关键点总结

|步骤|说明|
|---|---|
|hash 计算|`hashCode()` 高 16 位异或低 16 位|
|桶定位|`(n - 1) & hash` 位运算|
|key 判重|`hash` 相同 + `==` 或 `equals()`|
|链表插入|尾插法（JDK 8）|
|树化条件|链表长度 ≥ 8 且数组长度 ≥ 64|
|扩容触发|`++size > threshold`|

# HashMap 为什么使用红黑树（链表过长时转换）

## 问题背景

HashMap 在极端情况下（大量 hash 碰撞），某个桶的链表会很长，查询退化为 **O(n)**，严重影响性能。

## 红黑树 vs 链表

|维度|链表|红黑树|
|---|---|---|
|查找|O(n)|O(log n)|
|插入|O(1)（尾插）|O(log n)|
|删除|O(n)（查找）+ O(1)|O(log n)|
|内存|每个节点 2 个字段|每个节点多 parent + color（约 2 倍）|
|适用场景|元素少|元素多|

## 为什么选红黑树而不是其他树

|平衡树|优点|缺点|
|---|---|---|
|AVL 树|严格平衡，查找更快|插入/删除旋转多，维护成本高|
|**红黑树**|近似平衡，旋转次数少|查找略慢于 AVL|
|B 树/B+ 树|适合磁盘 IO|不适合内存场景|

红黑树是**折中选择**：

- 查找：O(log n) 足够好
- 插入/删除：旋转次数 ≤ 3，维护成本低
- 实际工程中被广泛验证（Linux、C++ STL、Java）

## 转换阈值的设计

```java
static final int TREEIFY_THRESHOLD = 8;    // 链表 → 红黑树
static final int UNTREEIFY_THRESHOLD = 6;  // 红黑树 → 链表
static final int MIN_TREEIFY_CAPACITY = 64; // 最小树化容量
```

### 为什么是 8 而不是更小？

根据泊松分布，当负载因子为 0.75 时：

|链表长度|出现概率|
|---|---|
|0|0.606|
|1|0.303|
|2|0.076|
|3|0.013|
|4|0.0016|
|5|0.00015|
|6|0.00001|
|7|0.0000006|
|**≥ 8**|**0.00000006**（千万分之一）|

链表长度达到 8 的概率极低，说明哈希质量很差或被恶意攻击。此时转为红黑树是"保底"策略。

### 为什么退化阈值是 6 而不是 8？

避免频繁转换：

- 如果进和出的阈值都是 8，元素增减频繁时会在链表和红黑树间反复转换
- 设为 6，中间留有缓冲区，减少树化和反树化的开销

### 为什么还需要数组长度 ≥ 64？

如果数组长度很小（如 16），链表长度达到 8 意味着利用率极高。此时**优先扩容**可以更有效地分散元素，比树化更划算。扩容后元素重新分布，链表大概率会变短。

## 转换流程

```
链表 → 红黑树（treeifyBin）：
  if (tab.length < MIN_TREEIFY_CAPACITY)
      resize();  // 先扩容
  else
      将链表转为红黑树

红黑树 → 链表（untreeify）：
  if (节点数 <= UNTREEIFY_THRESHOLD)
      将红黑树拆解为链表
```

# HashMap 的 key 为什么用 String / Integer

## 核心原因

作为 HashMap 的 key，需要满足两个条件：

1. **`hashCode()` 实现好**：哈希分布均匀，减少碰撞
2. **`equals()` 实现好**：能正确判断逻辑相等
3. **不可变（Immutable）**：`hashCode()` 在对象生命周期内不变

## String 作为 key

|优势|说明|
|---|---|
|不可变|String 一旦创建内容不可变，`hashCode()` 值固定不变|
|缓存 hashCode|String 内部缓存了 hash 值（`hash` 字段），首次计算后直接返回|
|equals 完善|String 重写了 `equals()`，按内容比较|
|常用场景|大部分场景的 key 本身就是字符串（用户名、ID 等）|

```java
// String 的 hashCode 缓存机制
public class String {
    private int hash;  // 默认 0，首次调用 hashCode() 时计算并缓存

    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;
            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;  // 缓存
        }
        return h;
    }
}
```

## Integer 作为 key

|优势|说明|
|---|---|
|不可变|Integer 是不可变类|
|hashCode 简单|直接返回 int 值本身，无碰撞|
|equals 简单|直接比较 int 值|

```java
// Integer 的 hashCode
public int hashCode() {
    return Integer.hashCode(value);  // 直接返回 value
}
```

## 为什么 key 必须不可变

如果 key 对象是可变的，修改了 key 的属性会导致：

1. `hashCode()` 返回值改变 → 找不到原来的桶
2. `equals()` 判断结果改变 → 无法匹配原节点

```java
// 危险示例
class MutableKey {
    int id;
    // 没有重写 hashCode 和 equals，或 hashCode 依赖可变字段

    public static void main(String[] args) {
        Map<MutableKey, String> map = new HashMap<>();
        MutableKey key = new MutableKey(1);
        map.put(key, "value");

        key.id = 2;  // 修改了 key 的属性
        map.get(key);  // null，找不到了！
    }
}
```

## 常用不可变类型作为 key

|类型|可用性|说明|
|---|---|---|
|String|✅ 最常用|不可变，hashCode 缓存|
|Integer / Long 等|✅ 常用|不可变，hashCode = 值|
|枚举|✅ 推荐|天然不可变，单例|
|自定义不可变类|✅ 需重写 hashCode + equals|所有字段 final|

# HashMap 遍历方式

## 三种遍历方式

### 1. entrySet（推荐）

```java
// 遍历键值对，最常用
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}

// Java 8 forEach
map.forEach((key, value) -> {
    System.out.println(key + ": " + value);
});
```

### 2. keySet

```java
// 先遍历 key，再通过 key 获取 value
for (String key : map.keySet()) {
    Integer value = map.get(key);
}

// Java 8
map.keySet().forEach(key -> {
    System.out.println(key + ": " + map.get(key));
});
```

### 3. values

```java
// 只遍历 value
for (Integer value : map.values()) {
    System.out.println(value);
}
```

## 性能对比

|方式|时间复杂度|说明|
|---|---|---|
|**entrySet**|O(n)|✅ 推荐，直接获取 key 和 value|
|keySet|O(n) + **额外查找**|每次 `get(key)` 需要 hash + 查找|
|values|O(n)|只需要 value 时使用|

### 为什么 entrySet 更好

```java
// keySet：遍历 + 二次查找
for (String key : map.keySet()) {
    Integer value = map.get(key);  // 每次都要重新 hash + 定位桶 + 遍历链表
}

// entrySet：遍历 + 直接获取
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();  // 直接从 Entry 获取，无需查找
}
```

- `keySet` 获取 key 后，`map.get(key)` 还要再做一次 hash 计算和桶定位
- `entrySet` 直接遍历 Entry 对象，已经包含了 key 和 value，**无需二次查找**

## 源码分析

```java
// keySet 返回的是 KeySet 视图
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}

// entrySet 返回的是 EntrySet 视图
public Set<Map.Entry<K, V>> entrySet() {
    Set<Map.Entry<K, V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

两者都是视图对象，遍历时直接访问底层数组，不会复制数据。

## 使用 Lambda/Stream

```java
// 遍历
map.forEach((k, v) -> System.out.println(k + "=" + v));

// filter + 收集
Map<String, Integer> filtered = map.entrySet().stream()
    .filter(e -> e.getValue() > 10)
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

// 按 value 排序
List<Map.Entry<String, Integer>> sorted = map.entrySet().stream()
    .sorted(Map.Entry.comparingByValue())
    .collect(Collectors.toList());
```

## 注意事项

1. 遍历时修改 Map 会抛 `ConcurrentModificationException`（fail-fast）
2. 只能用 `Iterator.remove()` 或 `entrySet().removeIf()` 安全删除
3. 遍历顺序不保证（HashMap 无序，LinkedHashMap 按插入序，TreeMap 按排序）

## 关联知识点

