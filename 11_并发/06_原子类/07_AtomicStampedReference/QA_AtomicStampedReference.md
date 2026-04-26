---
title: AtomicStampedReference
tags:
  - Java/并发
  - 问答
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicStampedReference（带版本号的对象引用，解决ABA问题）

## Q1：AtomicStampedReference 是怎么解决 ABA 问题的？

**A**：在 CAS 中增加版本号校验，同时比较引用和版本号：

```java
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

int stamp = ref.getStamp();  // 获取当前版本号
ref.compareAndSet(10, 20, stamp, stamp + 1);
//          期望引用 新引用  期望版本  新版本
```

即使值从 10→11→10（引用变化），版本号也从 0→1→2，CAS 会因为 stamp 不匹配而失败。

---

## Q2：AtomicStampedReference 内部是怎么实现的？

**A**：内部将引用和版本号打包为一个 `Pair` 对象，CAS 操作的是整个 `Pair` 的引用：

```java
private static class Pair<T> {
    final T reference;
    final int stamp;
}

private volatile Pair<V> pair;

public boolean compareAndSet(V expectedRef, V newRef,
                              int expectedStamp, int newStamp) {
    Pair<V> current = pair;
    return expectedRef == current.reference
        && expectedStamp == current.stamp
        && casPair(current, new Pair<>(newRef, newStamp));
}
```

CAS 保证 `Pair` 引用的原子替换，从而同时保证 reference 和 stamp 的一致性。

---

## Q3：版本号溢出了怎么办？

**A**：`stamp` 是 `int` 类型（-2^31 ~ 2^31-1）。实际场景中：

- 如果每次修改 stamp+1，溢出需要约 20 亿次操作
- 即使高并发（每秒百万次），也需要约 24 天才会溢出
- 溢出后变成负数，不影响正确性（只要 stamp 变过，旧的期望值就不匹配）
- 如果真的担心，可以用 `long` 类型的自定义实现，或定期重建 AtomicStampedReference

## 关联知识点
