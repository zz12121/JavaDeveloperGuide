---
title: LinkedHashMap 底层实现
tags:
  - Java/集合框架
  - 源码型
  - 问答
module: 05_集合框架
created: 2026-04-25
---

# LinkedHashMap 底层实现

## Q1：LinkedHashMap 和 HashMap 的核心区别是什么？

**A：**
LinkedHashMap 继承自 HashMap，在 HashMap 的"数组 + 链表/红黑树"结构基础上，额外维护了一条**双向链表**，用于记录元素的**插入顺序**或**访问顺序**。

关键区别：
1. LinkedHashMap.Entry 继承 HashMap.Node，新增了 `before` 和 `after` 两个指针
2. 遍历 LinkedHashMap 时按链表顺序输出，HashMap 遍历是无序的
3. LinkedHashMap 额外维护了 `head` 和 `tail` 两个指针指向链表的头尾节点

---

## Q2：LinkedHashMap 的 accessOrder 参数的作用是什么？

**A：**
`accessOrder` 控制元素的排序模式：
- `accessOrder = false`（默认）：**插入顺序**，每次 `put()` 新元素添加到链表末尾
- `accessOrder = true`：**访问顺序**，每次 `get()` 或 `put()` 都把访问的节点移动到链表末尾（LRU 模式）

```java
LinkedHashMap<String, Integer> map = new LinkedHashMap<>(16, 0.75f, true);
map.put("A", 1);
map.put("B", 2);
map.put("C", 3);
map.get("A");  // A 移到末尾
// 遍历顺序：B → C → A
```

---

## Q3：LinkedHashMap 如何实现 LRU 缓存？

**A：**
通过 `accessOrder=true` + `removeEldestEntry()` 实现：

```java
LinkedHashMap<Integer, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > 3;  // 超过3个元素时删除最老的
    }
};
// 每次 put() 后检查，超过容量时自动删除 head 节点
```

---

## Q4：HashMap.Node 和 LinkedHashMap.Entry 的区别是什么？

**A：**

| 字段 | HashMap.Node | LinkedHashMap.Entry |
|------|-------------|---------------------|
| before | 无 | 新增 |
| after | 无 | 新增 |
| 其他 | hash/key/value/next | 继承 |

LinkedHashMap.Entry 只新增两个指针，继承父类所有字段。

---

## Q5：LinkedHashMap 的 containsValue() 为什么比 HashMap 高效？

**A：**
HashMap 遍历所有桶，LinkedHashMap 重写了该方法，直接遍历双向链表：

```java
public boolean containsValue(Object value) {
    for (LinkedHashMap.Entry e = head; e != null; e = e.after) {
        if (Objects.equals(e.value, value)) return true;
    }
    return false;
}
```

性能更稳定，不受哈希分布影响。

---

## Q6：LinkedHashMap 是线程安全的吗？

**A：**
**不是**。与 HashMap 一样，多线程并发修改会导致数据结构破坏。解决方案：
- `Collections.synchronizedMap(new LinkedHashMap<>())`
- Guava Cache（推荐）

---

## Q7：LinkedHashMap 额外内存开销是多少？

**A：**
每个节点比 HashMap.Node 多两个指针（before/after），在 64 位 JVM 上多 16 字节。1,000,000 节点多约 16 MB 内存。

---

## Q8：LinkedHashMap 在 JDK 源码中重写了哪些方法？

**A：**
重写了 HashMap 的回调钩子方法：
- `newNode()`：插入时添加到链表末尾
- `afterNodeAccess()`：accessOrder=true 时移到末尾
- `afterNodeRemoval()`：删除时从链表摘除
- `containsValue()`：利用链表遍历

---

## Q9：MyBatis 一级缓存为什么用 LinkedHashMap？

**A：**
天然支持 LRU：`accessOrder=true` + `removeEldestEntry()` 自动淘汰最久未使用条目。MyBatis 设置 maxSize，超出时自动清理。
