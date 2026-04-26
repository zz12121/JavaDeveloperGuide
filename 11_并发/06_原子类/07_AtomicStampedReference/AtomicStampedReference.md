---
title: AtomicStampedReference
tags:
  - Java/并发
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# AtomicStampedReference（带版本号的对象引用，解决ABA问题）

## 先说结论

`AtomicStampedReference<V>` 通过在 CAS 中同时校验**对象引用 + 版本号（stamp）** 来解决 ABA 问题。每次修改时版本号自增，即使值回到原来的引用，版本号也已经不同，CAS 自然失败。

## 深度解析

### 内部结构

```java
public class AtomicStampedReference<V> {
    // 内部将 reference 和 stamp 打包成一个 Pair
    private static class Pair<T> {
        final T reference;  // 对象引用
        final int stamp;    // 版本号
        Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
    }

    private volatile Pair<V> pair;  // CAS 操作的原子单元

    // CAS 时同时比较引用和版本号
    public boolean compareAndSet(V expectedReference,
                                  V newReference,
                                  int expectedStamp,
                                  int newStamp) {
        Pair<V> current = pair;
        return expectedReference == current.reference
            && expectedStamp == current.stamp
            && ((newReference == current.reference && newStamp == current.stamp)
                || casPair(current, new Pair<>(newReference, newStamp)));
    }

    // 获取引用
    public V getReference() { return pair.reference; }
    // 获取版本号
    public int getStamp() { return pair.stamp; }
    // 同时获取引用和版本号
    public V get(int[] stampHolder) {
        stampHolder[0] = pair.stamp;
        return pair.reference;
    }
}
```

### ABA 解决原理

```
正常 CAS（无版本号）：
  T1: 读 ref=A
  T2: ref=A → ref=B → ref=A
  T1: CAS(A, C) ✓ ⚠️ 误判为未变

带版本号的 CAS：
  T1: 读 ref=A, stamp=0
  T2: CAS(A→B, stamp 0→1) → ref=B, stamp=1
  T2: CAS(B→A, stamp 1→2) → ref=A, stamp=2
  T1: CAS(ref=A, C, stamp=0, newStamp=1)
      → stamp 不匹配(0≠2) → 失败！✅ 正确检测到 ABA
```

### 核心方法

```java
// 构造：初始引用 + 初始版本号
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

// 获取（通过数组传出 stamp）
int[] stampHolder = new int[1];
Integer value = ref.get(stampHolder); // value=10, stampHolder[0]=0

// CAS 更新（同时校验引用和版本号）
boolean success = ref.compareAndSet(
    10,      // 期望引用
    20,      // 新引用
    0,       // 期望版本号
    1        // 新版本号
);

// 单独设置版本号（引用不变）
ref.set(20, 2); // ref=20, stamp=2

// 尝试设置版本号（CAS）
ref.attemptStamp(20, 3); // 如果当前引用==20，设置stamp=3
```

## 易错点/踩坑

- ❌ 版本号会自动递增——必须手动管理 stamp，每次 CAS 传入新版本号
- ❌ 认为 stamp 只能递增——stamp 可以是任意 int，不强制递增（但建议递增）
- ✅ 版本号可能溢出（int 范围 -2^31 ~ 2^31-1），但实际场景中几乎不可能

## 代码示例

```java
// 完整的 ABA 问题解决
public class StampedReferenceDemo {
    public static void main(String[] args) throws Exception {
        AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

        Thread t1 = new Thread(() -> {
            int stamp = ref.getStamp();
            System.out.println("t1: value=" + ref.getReference() + ", stamp=" + stamp);
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            boolean ok = ref.compareAndSet(10, 20, stamp, stamp + 1);
            System.out.println("t1 CAS: " + ok); // false — stamp 不匹配
        });

        Thread t2 = new Thread(() -> {
            int stamp = ref.getStamp();
            ref.compareAndSet(10, 11, stamp, stamp + 1); // stamp: 0→1
            stamp = ref.getStamp();
            ref.compareAndSet(11, 10, stamp, stamp + 1); // stamp: 1→2
            System.out.println("t2: ABA发生, stamp=" + ref.getStamp());
        });

        t1.start(); t2.start();
        t1.join();  t2.join();
    }
}
```

## 关联知识点
