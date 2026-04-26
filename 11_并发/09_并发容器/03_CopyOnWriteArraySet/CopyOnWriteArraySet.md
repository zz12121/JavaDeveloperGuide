---
title: CopyOnWriteArraySet
tags:
  - Java/并发
  - 原理型
module: 09_并发容器
created: 2026-04-18
---

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

## 关联知识点

