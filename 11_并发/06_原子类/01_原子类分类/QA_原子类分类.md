---
title: 原子类分类
tags:
  - Java/并发
  - 问答
  - 原理型
module: 06_原子类
created: 2026-04-18
---

# 原子类分类（基本类型/数组类型/引用类型/字段更新器）

## Q1：JUC 原子类有哪些分类？

**A**：四大类：

1. **基本类型**：`AtomicInteger`、`AtomicLong`、`AtomicBoolean`
2. **数组类型**：`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`
3. **引用类型**：`AtomicReference`、`AtomicStampedReference`、`AtomicMarkableReference`
4. **字段更新器**：`AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater`

另外 JDK8+ 新增了累加器：`LongAdder`、`LongAccumulator`、`DoubleAdder`、`DoubleAccumulator`。

---

## Q2：原子类的底层实现原理是什么？

**A**：所有原子类基于三个核心机制：

1. **volatile**：修饰值字段，保证多线程之间的可见性
2. **Unsafe**：通过字段偏移量直接操作内存
3. **CAS**：硬件级原子指令（`cmpxchg`），保证 compare-and-swap 的原子性

```java
// 以 AtomicInteger 为例
private volatile int value;                          // volatile 可见性
private static final long VALUE = U.objectFieldOffset(...); // 偏移量
U.compareAndSetInt(this, VALUE, expected, newValue); // CAS 原子性
```

---

## Q3：字段更新器和直接用原子类有什么区别？

**A**：

| 维度 | 原子类 | 字段更新器 |
|------|--------|-----------|
| 内存开销 | 每个字段一个 AtomicXxx 对象 | 不创建额外对象 |
| 使用方式 | 包装字段为原子对象 | 直接操作对象字段 |
| 限制 | 无 | 字段必须是 volatile |
| 适用场景 | 新代码 | 已有对象，不想改结构 |

字段更新器适合对已有类的 volatile 字段做原子更新，避免额外的对象包装开销。

---

## Q4：AtomicInteger vs AtomicLong vs LongAdder 怎么选？

**A**：

| 场景 | 推荐 | 原因 |
|------|------|------|
| 普通计数器（int 范围） | AtomicInteger | 内存小，API 丰富 |
| 需要精确 long 值返回 | AtomicLong | 支持 sum() 精确聚合 |
| 高并发计数器 | LongAdder | 分段 CAS，并发下性能 ~10x |
| 序号生成（需要 CAS） | AtomicLong | LongAdder 不支持精确 compareAndSet |

```java
// ❌ 错误：高并发下 AtomicLong 的 CAS 竞争激烈
AtomicLong counter = new AtomicLong(0);
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> counter.incrementAndGet());
}

// ✅ 正确：高并发用 LongAdder
LongAdder counter = new LongAdder();
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> counter.increment());
}

// ⚠️ 注意：LongAdder 的 sum() 不是原子快照，
// 极端并发下可能有轻微误差（~0.0001%）
```

---

## Q5：AtomicStampedReference vs AtomicMarkableReference 怎么选？

**A**：两者都是解决 ABA 问题，区别在于携带的额外信息：

| 类 | 额外信息 | 典型应用 |
|----|----------|----------|
| AtomicStampedReference | int 版本号（每次 CAS 递增） | 乐观重试、版本控制 |
| AtomicMarkableReference | boolean 标记位 | 节点删除标记、一次性对象 |

```java
// AtomicStampedReference：用于栈的 CAS
AtomicStampedReference<Node> stack = new AtomicStampedReference<>(top, 0);
int[] stamp = new int[1];
Node cur = stack.get(stamp);       // 获取当前引用+版本号
stack.compareAndSet(cur, newNode, stamp[0], stamp[0] + 1);

// AtomicMarkableReference：用于对象删除标记
AtomicMarkableReference<User> users = new AtomicMarkableReference<>(user, false);
users.compareAndSet(user, updatedUser, false, false); // 标记位不变
users.compareAndSet(user, null, false, true);        // 标记为已删除
```

---

## Q6：AtomicReferenceArray 和 AtomicReference 有什么区别？

**A**：

| 维度 | AtomicReference | AtomicReferenceArray |
|------|----------------|---------------------|
| 操作对象 | 单个对象引用 | 数组中的对象引用 |
| 索引方式 | 无索引 | 整数索引 |
| CAS 范围 | 整个引用 | 数组指定索引的元素 |
| 典型场景 | 单对象乐观锁 | 无锁队列、连接池 |

```java
// AtomicReference：单对象
AtomicReference<Order> currentOrder = new AtomicReference<>();
currentOrder.compareAndSet(oldOrder, newOrder);

// AtomicReferenceArray：数组元素
AtomicReference<Order>[] orders = new AtomicReferenceArray<>(100);
// 无锁队列中：HEAD = orders[headIndex], TAIL = orders[tailIndex]
```

---

## Q7：什么时候用字段更新器而不是原子类？

**A**：两个核心场景：

1. **已有类不想改结构**（减少对象数量）：
```java
// ❌ 每个 User 创建 AtomicInteger
class User1 {
    AtomicInteger score = new AtomicInteger(0); // 每个 User 多一个对象
}

// ✅ 用 Updater，不创建额外对象
class User2 {
    volatile int score; // 直接是普通字段
}
AtomicIntegerFieldUpdater<User2> UPDATER =
    AtomicIntegerFieldUpdater.newUpdater(User2.class, "score");
```

2. **大批量对象的高效统计**：
```java
// 1万个 User，用 AtomicInteger 需要创建 1万个对象
// 用 Updater 只需一个 Updater 实例
private static final AtomicIntegerFieldUpdater<User> SCORE_UPDATER =
    AtomicIntegerFieldUpdater.newUpdater(User.class, "score");
```

## 关联知识点

