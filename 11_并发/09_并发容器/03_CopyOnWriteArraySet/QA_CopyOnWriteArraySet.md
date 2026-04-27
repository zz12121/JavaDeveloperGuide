
# CopyOnWriteArraySet QA

## Q1: CopyOnWriteArraySet 和 ConcurrentSkipListSet 如何选择？

**A:**

| 维度 | CopyOnWriteArraySet | ConcurrentSkipListSet |
|------|---------------------|----------------------|
| 底层 | CopyOnWriteArrayList（数组） | ConcurrentSkipListMap（跳表） |
| 有序性 | ❌ 无序 | ✅ 按自然顺序 |
| add 复杂度 | O(n) | O(log n) |
| contains 复杂度 | O(n) | O(log n) |
| 迭代一致性 | 快照（弱一致） | 无锁遍历（弱一致） |
| 内存开销 | 写时复制（高） | 索引节点（中等） |

**选择建议：**
- 需要有序 → `ConcurrentSkipListSet`
- 小数据量 + 读多写少 → `CopyOnWriteArraySet`
- 大数据量 → `ConcurrentSkipListSet`

---

## Q2: CopyOnWriteArraySet 能存 null 吗？

**A:**

`CopyOnWriteArraySet` 底层是 `CopyOnWriteArrayList`，但 Set 不允许存 null（和 HashSet 一样）。

```java
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
set.add(null);  // ❌ NullPointerException（和 HashSet 行为一致）
```

**原因**：Set 的语义要求元素唯一且非 null，与 HashSet 保持一致。

---

## Q3: CopyOnWriteArraySet 的迭代器支持 remove 吗？

**A:**

不支持。`CopyOnWriteArraySet` 的迭代器继承自 `CopyOnWriteArrayList.COWIterator`，是快照迭代器，不支持 remove/set/add 操作。

```java
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
set.add("A"); set.add("B");

Iterator<String> iter = set.iterator();
iter.remove();  // ❌ UnsupportedOperationException
```

**正确做法**：在 Set 本身上调用 remove。
```java
Iterator<String> iter = set.iterator();
while (iter.hasNext()) {
    if (iter.next().equals("A")) {
        set.remove("A");  // ✅ 在 set 上删除
    }
}
```

---

## Q4: CopyOnWriteArraySet 如何保证线程安全？

**A:**

通过 `CopyOnWriteArrayList.addIfAbsent()` 保证去重，底层使用：
1. **ReentrantLock**：写操作加锁
2. **volatile 数组引用**：保证可见性
3. **Arrays.copyOf**：写时复制新数组

```java
public boolean add(E e) {
    return al.addIfAbsent(e);  // 内部 ReentrantLock + 数组复制
}
```

---

## Q5: 什么场景下 CopyOnWriteArraySet 比 ConcurrentSkipListSet 更合适？

**A:**

1. **元素数量很小**（< 100）：COWAS 的 O(n) 常数因子更小
2. **读多写极少**：写时复制开销可接受
3. **不需要有序**：不需要 Navigable API
4. **迭代频繁**：快照迭代器提供更稳定的遍历

**典型场景：**
- 配置列表（如黑白名单）
- 事件监听器列表
- 小型注册表

