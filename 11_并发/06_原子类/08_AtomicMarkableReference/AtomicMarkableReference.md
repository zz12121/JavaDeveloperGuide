---
title: AtomicMarkableReference
tags:
  - Java/并发
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicMarkableReference（带标记位的对象引用）

## 先说结论

`AtomicMarkableReference<V>` 通过在 CAS 中同时校验**对象引用 + 布尔标记位（mark）** 来解决 ABA 问题。与 `AtomicStampedReference` 的区别是：标记位只有 `true/false` 两个状态，适用于只需要知道"是否被修改过"的场景，如垃圾回收标记、节点删除标记等。

## 深度解析

### 内部结构

```java
public class AtomicMarkableReference<V> {
    private static class Pair<T> {
        final T reference;
        final boolean mark;
        Pair(T reference, boolean mark) {
            this.reference = reference;
            this.mark = mark;
        }
    }

    private volatile Pair<V> pair;

    public boolean compareAndSet(V expectedReference,
                                  V newReference,
                                  boolean expectedMark,
                                  boolean newMark) {
        Pair<V> current = pair;
        return expectedReference == current.reference
            && expectedMark == current.mark
            && ((newReference == current.reference && newMark == current.mark)
                || casPair(current, new Pair<>(newReference, newMark)));
    }

    public V getReference() { return pair.reference; }
    public boolean isMarked() { return pair.mark; }
    public V get(boolean[] markHolder) {
        markHolder[0] = pair.mark;
        return pair.reference;
    }
}
```

### AtomicStampedReference vs AtomicMarkableReference

| 维度 | AtomicStampedReference | AtomicMarkableReference |
|------|----------------------|------------------------|
| 附加信息 | int 版本号（stamp） | boolean 标记位（mark） |
| 状态数 | 2^32 种 | 2 种（true/false） |
| 检测能力 | 精确到第几次修改 | 只知道"改没改过" |
| 适用场景 | 需要完整修改历史 | 只需"脏标记" |
| 典型应用 | 链表 CAS、版本控制 | GC 标记、节点删除 |

### 典型应用：标记删除

```java
// 无锁链表的标记删除
class Node<T> {
    T value;
    AtomicMarkableReference<Node<T>> next;
    Node(T value) {
        this.value = value;
        this.next = new AtomicMarkableReference<>(null, false);
    }
}

// 删除节点：先标记 mark=true，再 CAS 移除
boolean delete(Node<T> prev, Node<T> node) {
    // 第一步：标记
    boolean[] markHolder = {false};
    Node<T> succ = node.next.get(markHolder);
    if (!node.next.compareAndSet(succ, succ, false, true)) {
        return false; // 已被其他线程标记
    }
    // 第二步：CAS 移除
    return prev.next.compareAndSet(node, succ, false, false);
}
```

## 易错点/踩坑

- ❌ 认为 mark 可以存储更多信息——只有 true/false 两种状态
- ❌ 忽略 mark 的语义——true 通常表示"已删除"/"已修改"/"脏"
- ✅ 标记删除是 mark 的经典用法：先 mark 再 CAS 移除，保证删除的安全性

## 代码示例

```java
public class MarkableRefDemo {
    public static void main(String[] args) {
        AtomicMarkableReference<String> ref =
            new AtomicMarkableReference<>("initial", false);

        // 修改并标记
        boolean[] markHolder = {false};
        String value = ref.get(markHolder);
        System.out.println("value=" + value + ", marked=" + markHolder[0]);

        // CAS：值 + 标记一起更新
        ref.compareAndSet("initial", "modified", false, true);

        value = ref.get(markHolder);
        System.out.println("value=" + value + ", marked=" + markHolder[0]);
        // value=modified, marked=true

        // 尝试不带标记更新（失败，因为 mark 不匹配）
        boolean ok = ref.compareAndSet("modified", "new", false, false);
        System.out.println("CAS without mark: " + ok); // false
    }
}
```

## 关联知识点