
# CopyOnWriteArraySet

## 核心结论

CopyOnWriteArraySet 基于 CopyOnWriteArrayList 实现，使用 `addIfAbsent` 保证元素唯一性。本质是一个线程安全的 Set，底层是数组，没有 HashMap 的桶结构。

## 深度解析

### 源码

```java
public class CopyOnWriteArraySet<E> implements Set<E> {
    private final CopyOnWriteArrayList<E> al;

    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }

    public boolean add(E e) {
        return al.addIfAbsent(e); // 核心：不存在才添加
    }
}
```

### addIfAbsent 实现

```java
// CopyOnWriteArrayList.addIfAbsent
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray(); // volatile 读快照
    return indexOf(e, snapshot, 0, snapshot.length) >= 0
        ? false                      // 已存在，返回 false
        : addIfAbsent(e, snapshot);  // 不存在，尝试添加
}

private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        // 双重检查：加锁后再确认不存在
        if (snapshot != current) {
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                return false;
        }
        // 复制新数组并添加
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 特性

| 维度 | 说明 |
|------|------|
| 底层 | CopyOnWriteArrayList |
| 去重 | addIfAbsent 遍历检查 |
| 读 | 无锁 |
| 写 | 锁 + 复制数组 |
| 一致性 | 弱一致 |
| contains | O(n) 遍历 |

### 性能问题

- `addIfAbsent` 每次 add 都要遍历数组检查是否存在，O(n)
- `contains` 也是 O(n) 遍历
- 不适合大量元素的 Set 场景

### 适用场景

- 元素较少的 Set（如注册表、监听器去重）
- 读多写少
- 不需要高效 contains 查询

## 易错点与踩坑

### 1. addIfAbsent 的双重检查锁造成性能浪费

```java
// ❌ 每次 add 都要遍历数组检查是否存在
public boolean add(E e) {
    Object[] snapshot = getArray();  // 快照读
    // 第一次遍历：检查是否存在
    if (indexOf(e, snapshot, 0, snapshot.length) >= 0) {
        return false;  // 存在，直接返回
    }
    // 不存在，尝试添加（可能失败）
    return addIfAbsent(e, snapshot);
}

// 1000 个元素，每次 add 要遍历 1000 次
// 100 次 add = 最多 100 × 1000 = 100000 次比较
// ✅ 如果元素较多，用 ConcurrentSkipListSet（O(log n)）
```

### 2. contains 方法也是 O(n)，容易被误认为 O(1)

```java
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
// 添加 10000 个元素

// ❌ 误以为 contains 是 O(1)
boolean exists = set.contains("target");  // 实际遍历 10000 次

// ✅ 数据量大时，contains 性能很差
// ✅ 如果需要高效 contains，用 HashSet 相关（但不是线程安全）
```

### 3. 与 ConcurrentSkipListSet 的选择错误

```java
// ❌ 场景：需要去重 + 有序遍历
CopyOnWriteArraySet<Integer> cowas = new CopyOnWriteArraySet<>();
cowas.add(5);
cowas.add(1);
cowas.add(3);
// 遍历顺序：5, 1, 3（无序！）

// ✅ 正确选择
ConcurrentSkipListSet<Integer> csls = new ConcurrentSkipListSet<>();
csls.add(5);
csls.add(1);
csls.add(3);
// 遍历顺序：1, 3, 5（有序）

// ⚠️ COWAS 虽然底层是数组，但没有排序逻辑
```

## 关联知识点

