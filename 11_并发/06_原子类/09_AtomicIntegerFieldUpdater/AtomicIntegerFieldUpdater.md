---
title: AtomicIntegerFieldUpdater
tags:
  - Java/并发
  - 场景型
module: 06_原子类
created: 2026-04-18
---

# AtomicIntegerFieldUpdater（原子更新对象字段，减少内存开销）

## 先说结论

`AtomicIntegerFieldUpdater<T>` 是一种基于**反射 + Unsafe** 的原子字段更新器，可以直接对已有对象的 `volatile int` 字段进行 CAS 操作，无需将字段包装为 `AtomicInteger` 对象，从而减少内存开销。适合不需要额外创建对象的场景。

## 深度解析

### 使用方式

```java
public class User {
    volatile int age;  // 必须是 volatile int！
}

// 创建 Updater（通过反射获取字段偏移量）
AtomicIntegerFieldUpdater<User> AGE_UPDATER =
    AtomicIntegerFieldUpdater.newUpdater(User.class, "age");

User user = new User();
AGE_UPDATER.set(user, 25);
AGE_UPDATER.incrementAndGet(user);  // age: 25 → 26
AGE_UPDATER.compareAndSet(user, 26, 30); // age: 26 → 30
```

### 底层原理

```java
// 内部基于 Unsafe + 反射
public abstract class AtomicIntegerFieldUpdater<T> {
    // 反射获取字段的偏移量
    // CAS 操作通过 Unsafe.compareAndSetInt(obj, offset, expect, update)
}

// 实现类：CASFieldUpdater（基于 Unsafe）
private static final class CASFieldUpdater<T>
    extends AtomicIntegerFieldUpdater<T> {
    private final long offset;
    CASFieldUpdater(Class<T> tClass, String fieldName) {
        Field field = tClass.getDeclaredField(fieldName);
        offset = U.objectFieldOffset(field);
    }
    public boolean compareAndSet(T obj, int expect, int update) {
        return U.compareAndSetInt(obj, offset, expect, update);
    }
}
```

### 三种 Updater

```java
// int 字段
AtomicIntegerFieldUpdater<User> intUpdater =
    AtomicIntegerFieldUpdater.newUpdater(User.class, "age");

// long 字段
AtomicLongFieldUpdater<Account> longUpdater =
    AtomicLongFieldUpdater.newUpdater(Account.class, "balance");

// 引用字段
AtomicReferenceFieldUpdater<Node, Node> refUpdater =
    AtomicReferenceFieldUpdater.newUpdater(Node.class, Node.class, "next");
```

### 使用限制

```
限制条件：
✅ 字段必须是 volatile
✅ 字段必须是 int/long/引用类型
✅ 字段不能是 private（对 Updater 所在类不可见）
✅ 类必须有 Updater 能访问到的字段

❌ 字段不能是 final（final 不允许 CAS）
❌ 字段不能是非 volatile（CAS 需要 volatile 语义）
❌ 不支持静态字段（只能操作实例字段）
```

### vs 直接使用 AtomicInteger

| 维度 | AtomicInteger | AtomicIntegerFieldUpdater |
|------|--------------|-------------------------|
| 内存开销 | 每个 16 bytes（对象头+value） | 几乎零额外开销 |
| 使用方式 | 包装字段为独立对象 | 直接操作原始字段 |
| 可读性 | ✅ 更直观 | ⚠️ 反射式调用 |
| 性能 | 略好（直接偏移量） | 略差（反射初始化，运行时一样） |
| 适用场景 | 新代码、字段独立 | 已有类、不想改结构 |

## 易错点/踩坑

- ❌ 忘记字段加 `volatile`——运行时抛 `IllegalArgumentException`
- ❌ 字段是 `private` 且 Updater 不在同一包——反射无法访问
- ✅ Updater 创建是一次性的，后续 CAS 操作性能与 AtomicInteger 相当

## 代码示例

```java
// 实际应用：统计每个用户的请求次数
public class RequestCounter {
    private volatile int count;

    private static final AtomicIntegerFieldUpdater<RequestCounter> COUNT_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(RequestCounter.class, "count");

    public void increment() {
        COUNT_UPDATER.incrementAndGet(this);
    }

    public int getCount() {
        return COUNT_UPDATER.get(this);
    }
}

// 与 AtomicInteger 对比内存开销
// 方案1：每个用户一个 AtomicInteger 对象 → 大量小对象，GC 压力
// 方案2：用 Updater 直接操作字段 → 无额外对象
```

## 关联知识点
