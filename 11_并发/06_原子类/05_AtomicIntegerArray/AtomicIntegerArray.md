---
title: AtomicIntegerArray
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicIntegerArray（数组元素的原子更新）

## 先说结论

`AtomicIntegerArray` 提供对 `int[]` 数组中**单个元素**的原子操作。注意：它只保证单个数组元素的原子更新，不保证多个元素操作的原子性。底层通过计算 `baseOffset + index * scale` 定位元素内存地址，然后用 CAS 更新。

## 深度解析

### 核心源码

```java
public class AtomicIntegerArray implements java.io.Serializable {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final int ABASE = U.arrayBaseOffset(int[].class);  // 数组头偏移
    private static final int ASHIFT = 31 - Integer.numberOfLeadingZeros(
        U.arrayIndexScale(int[].class));  // 元素大小位移

    private final int[] array;  // 内部持有数组引用（非 volatile）

    // 通过 index 计算内存偏移量
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length) throw new IndexOutOfBoundsException();
        return byteOffset(i);
    }

    private static long byteOffset(int i) {
        return (long) i << ASHIFT + ABASE;
    }

    // 原子获取
    public final int get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    // 原子设置
    public final void set(int i, int newValue) {
        U.putIntVolatile(array, checkedByteOffset(i), newValue);
    }

    // CAS 更新
    public final boolean compareAndSet(int i, int expected, int newValue) {
        return U.compareAndSetInt(array, checkedByteOffset(i), expected, newValue);
    }

    // 原子自增
    public final int incrementAndGet(int i) {
        return U.getAndAddInt(array, checkedByteOffset(i), 1) + 1;
    }
}
```

### 内存定位原理

```
int[] array = new int[5];

内存布局：
┌────────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ 数组头 │ [0]  │ [1]  │ [2]  │ [3]  │ [4]  │      │
│ 16字节 │      │      │      │      │      │      │
└────────┴──────┴──────┴──────┴──────┴──────┴──────┘
         ↑
        ABASE  ABASE+4  ABASE+8  ...

byteOffset(2) = ABASE + 2 * 4 = ABASE + 8
CAS 通过这个偏移量直接定位到 array[2] 的内存地址
```

### 常用方法

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `get(i)` | 获取元素 | 当前值 |
| `set(i, v)` | 设置元素 | void |
| `lazySet(i, v)` | 延迟设置 | void |
| `getAndSet(i, v)` | 设置并返回旧值 | 旧值 |
| `compareAndSet(i, e, v)` | CAS 更新 | boolean |
| `getAndIncrement(i)` | array[i]++ | 旧值 |
| `incrementAndGet(i)` | ++array[i] | 新值 |
| `getAndAdd(i, d)` | array[i] += d | 旧值 |
| `addAndGet(i, d)` | array[i] += d | 新值 |

## 易错点/踩坑

- ❌ 认为多个数组元素操作是原子的——只保证单个元素的操作原子性
- ❌ 数组本身不是线程安全的——`array.length` 或修改数组引用不在保护范围
- ✅ AtomicIntegerArray 内部持有数组引用，外部不应直接访问该数组

## 代码示例

```java
// 并发直方图统计
public class ConcurrentHistogram {
    private final AtomicIntegerArray buckets;

    public ConcurrentHistogram(int bucketCount) {
        buckets = new AtomicIntegerArray(bucketCount);
    }

    public void record(int value) {
        int bucket = Math.min(value, buckets.length() - 1);
        buckets.incrementAndGet(bucket);
    }

    public int[] getSnapshot() {
        int[] result = new int[buckets.length()];
        for (int i = 0; i < result.length; i++) {
            result[i] = buckets.get(i);
        }
        return result;
    }
}
```

### AtomicLongArray / AtomicReferenceArray 对比

```java
// AtomicLongArray：long[] 元素的原子操作
AtomicLongArray arr = new AtomicLongArray(10);
arr.set(0, 100L);
arr.getAndAdd(0, 5);                     // 100 → 105
arr.compareAndSet(0, 105, 200);          // 105 → 200

// AtomicReferenceArray<E>：引用数组的原子操作
AtomicReferenceArray<String> refArr = new AtomicReferenceArray<>(5);
refArr.set(0, "hello");
refArr.compareAndSet(0, "hello", "world"); // CAS 替换
```

| 维度 | AtomicIntegerArray | AtomicLongArray | AtomicReferenceArray |
|------|-------------------|-----------------|---------------------|
| 元素类型 | int | long | E（泛型引用） |
| API | getAndIncrement 等 | 同左 | compareAndSet / getAndSet |
| 索引方式 | 整数索引 | 整数索引 | 整数索引 |
| 适用场景 | 计数器数组 | 数量数组 | 对象引用数组 |

> **注意**：`AtomicReferenceArray` 的泛型类型在运行时被擦除，但数组元素本身仍然是类型安全的 CAS。

## 关联知识点
