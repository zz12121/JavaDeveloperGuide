---
title: AtomicInteger
tags:
  - Java/并发
  - 源码型
module: 06_原子类
created: 2026-04-18
---

# AtomicInteger（getAndIncrement/incrementAndGet/decrementAndGet/getAndAdd/addAndGet）

## 先说结论

`AtomicInteger` 是最常用的原子类，通过 `Unsafe` + `volatile` + CAS 实现整数类型的无锁原子操作。支持自增、自减、加法、CAS 更新等操作，所有方法都是原子的，但复合操作（如 check-then-act）仍需额外同步。

## 深度解析

### 核心源码

```java
public class AtomicInteger implements java.io.Serializable {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
    private volatile int value;

    // 获取值
    public final int get() { return value; }

    // 设置值（volatile 写）
    public final void set(int newValue) { value = newValue; }

    // 延迟设置（不保证立即可见，但最终可见）
    public final void lazySet(int newValue) {
        U.putOrderedInt(this, VALUE, newValue);
    }

    // CAS 更新
    public final boolean compareAndSet(int expected, int newValue) {
        return U.compareAndSetInt(this, VALUE, expected, newValue);
    }

    // 原子自增，返回旧值（i++）
    public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }

    // 原子自增，返回新值（++i）
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }

    // 原子自减，返回旧值（i--）
    public final int getAndDecrement() {
        return U.getAndAddInt(this, VALUE, -1);
    }

    // 原子自减，返回新值（--i）
    public final int decrementAndGet() {
        return U.getAndAddInt(this, VALUE, -1) - 1;
    }

    // 原子加法，返回旧值
    public final int getAndAdd(int delta) {
        return U.getAndAddInt(this, VALUE, delta);
    }

    // 原子加法，返回新值
    public final int addAndGet(int delta) {
        return U.getAndAddInt(this, VALUE, delta) + delta;
    }
}
```

### Unsafe.getAndAddInt 源码

```java
// JDK9+ jdk.internal.misc.Unsafe
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // volatile 读当前值
    } while (!compareAndSwapInt(         // CAS 自旋更新
        o, offset, v, v + delta
    ));
    return v;  // 返回旧值
}
```

### 方法对比

| 方法 | 操作 | 返回值 | 等价 |
|------|------|--------|------|
| `getAndIncrement()` | +1 | 旧值 | i++ |
| `incrementAndGet()` | +1 | 新值 | ++i |
| `getAndDecrement()` | -1 | 旧值 | i-- |
| `decrementAndGet()` | -1 | 新值 | --i |
| `getAndAdd(n)` | +n | 旧值 | tmp=i; i+=n; return tmp |
| `addAndGet(n)` | +n | 新值 | i+=n; return i |
| `getAndSet(n)` | =n | 旧值 | tmp=i; i=n; return tmp |
| `compareAndSet(a,b)` | CAS | boolean | if(i==a) { i=b; return true } |

## 易错点/踩坑

- ❌ 认为 `if (ai.get() == 0) ai.set(1)` 是原子的——get 和 set 之间可能被其他线程修改
- ❌ 认为 `set()` 和 `lazySet()` 没区别——`lazySet()` 使用 `putOrderedInt`，延迟刷新到主存
- ✅ 复合操作需要用 `compareAndSet` 循环或 `accumulateAndGet`

## 代码示例

```java
// 线程安全计数器
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public int increment() { return count.incrementAndGet(); }
    public int get() { return count.get(); }

    // 复合操作：CAS + 循环
    public int update(int oldValue, int newValue) {
        return count.compareAndSet(oldValue, newValue) ? newValue : count.get();
    }

    // 函数式更新（JDK8+）
    public int multiply(int factor) {
        return count.updateAndGet(x -> x * factor);
    }

    // 累加器（JDK8+）
    public int accumulate(int delta) {
        return count.accumulateAndGet(delta, Integer::sum);
    }
}
```

## 关联知识点
