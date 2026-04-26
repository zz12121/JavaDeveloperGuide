---
title: AtomicIntegerArray
tags:
  - Java/并发
  - 场景型
  - 问答
module: 06_原子类
created: 2026-04-18
---

# AtomicIntegerArray（数组元素的原子更新）

## Q1：AtomicIntegerArray 是怎么定位数组元素的？

**A**：通过 Unsafe 计算数组元素的内存偏移量：

```java
static final int ABASE = U.arrayBaseOffset(int[].class);  // 数组对象头大小
static final int ASHIFT = 31 - Integer.numberOfLeadingZeros(
    U.arrayIndexScale(int[].class));  // 元素大小的位移

// index 的内存偏移 = ABASE + index * 4（int 占 4 字节）
long byteOffset(int i) {
    return (long) i << ASHIFT + ABASE;
}
```

CAS 通过 `compareAndSetInt(array, byteOffset(i), expected, newValue)` 直接对指定内存地址操作。

---

## Q2：AtomicIntegerArray 能保证多个元素同时更新的原子性吗？

**A**：**不能**。`AtomicIntegerArray` 只保证**单个元素**的操作是原子的：

```java
// ❌ 非原子操作
arr.compareAndSet(0, old0, new0);  // 元素0
arr.compareAndSet(1, old1, new1);  // 元素1
// 两个 CAS 之间可能被其他线程修改

// ✅ 如果需要同时更新多个元素，仍需加锁
synchronized (lock) {
    arr.set(0, new0);
    arr.set(1, new1);
}
```

---

## Q3：AtomicIntegerArray 和 volatile int[] 有什么区别？

**A**：

| 维度 | volatile int[] | AtomicIntegerArray |
|------|---------------|-------------------|
| 数组引用可见性 | ✅ volatile | N/A（包装类内部持有） |
| 元素修改可见性 | ❌ | ✅ putIntVolatile |
| 元素 CAS | ❌ | ✅ |
| 元素 i++ | ❌ | ✅ |
| 适用场景 | 数组引用偶尔变 | 数组元素频繁并发修改 |

`volatile int[]` 只保证数组引用的可见性，不保证数组内容修改的可见性。

---

## Q4：AtomicIntegerArray vs AtomicLongArray vs AtomicReferenceArray 的区别？

**A**：

| 维度 | AtomicIntegerArray | AtomicLongArray | AtomicReferenceArray |
|------|-------------------|-----------------|---------------------|
| 元素类型 | int | long | E（泛型引用） |
| CAS 方法 | compareAndSet(int) | compareAndSet(long) | compareAndSet(E) |
| 累加方法 | getAndAdd / incrementAndGet | 同左 | 无（引用类型不支持） |
| 适用场景 | 计数器、整数统计 | 大数值统计 | 对象引用替换 |

```java
// 选型决策：
// int 计数器 → AtomicIntegerArray
// long 计数器 → AtomicLongArray
// 对象引用 → AtomicReferenceArray
```

---

## Q5：AtomicIntegerArray 的 lazySet 有什么用？

**A**：`lazySet` 是比普通 `set` 更轻量的写操作，不保证其他线程立即看到新值：

```java
// set：volatile 写（StoreStore + StoreLoad 屏障）
arr.set(i, value);

// lazySet：普通写（无屏障，仅 StoreStore）
// 后续的 volatile 写会刷掉之前的数据
arr.lazySet(i, value); // 性能更好，但不保证可见性
```

**使用场景**：
- 不需要立即可见的场景（如初始化后不再读取的临时值）
- 高频写入、读取频率低的场景
- 相当于 `Unsafe.putOrderedInt`，性能优于 volatile 写

## 关联知识点
