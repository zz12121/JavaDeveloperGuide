---
title: AtomicInteger
tags:
  - Java/并发
  - 问答
  - 源码型
module: 06_原子类
created: 2026-04-18
---

# AtomicInteger（getAndIncrement/incrementAndGet/decrementAndGet/getAndAdd/addAndGet）

## Q1：AtomicInteger 的 incrementAndGet 底层怎么实现的？

**A**：通过 `Unsafe.getAndAddInt` + 自旋 CAS：

```java
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}

// Unsafe.getAndAddInt
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // ① volatile 读当前值
    } while (!compareAndSwapInt(         // ② CAS 更新
        o, offset, v, v + delta
    ));
    return v;  // 返回旧值
}
```

`getAndAddInt` 返回旧值，所以 `incrementAndGet = getAndAddInt(...) + 1` 返回新值。

---

## Q2：getAndIncrement 和 incrementAndGet 有什么区别？

**A**：

| 方法 | 等价 | 返回值 |
|------|------|--------|
| `getAndIncrement()` | `i++` | 自增前的旧值 |
| `incrementAndGet()` | `++i` | 自增后的新值 |

```java
AtomicInteger ai = new AtomicInteger(10);
int old = ai.getAndIncrement();  // old=10, ai=11
int newVal = ai.incrementAndGet(); // newVal=12, ai=12
```

---

## Q3：AtomicInteger 的复合操作怎么保证原子性？

**A**：单个方法是原子的，但多个方法的组合不是：

```java
// ❌ 非原子
if (ai.get() == 0) ai.set(1);  // get 和 set 之间可能被修改

// ✅ 方案1：CAS 循环
int old;
do {
    old = ai.get();
} while (!ai.compareAndSet(old, 1));

// ✅ 方案2：JDK8+ 函数式更新
ai.updateAndGet(x -> x == 0 ? 1 : x);
```

## 关联知识点

