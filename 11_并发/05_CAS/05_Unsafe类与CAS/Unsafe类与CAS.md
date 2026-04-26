---
title: Unsafe类与CAS
tags:
  - Java/并发
  - 源码型
module: 05_CAS
created: 2026-04-18
---

# Unsafe类与CAS（sun.misc.Unsafe，底层CAS实现，直接操作内存）

## 先说结论

`Unsafe` 是 JDK 提供的底层操作类，封装了直接内存操作、CAS、线程调度等 native 方法。它是所有原子类和 JUC 框架的底层依赖。JDK9 后从 `sun.misc.Unsafe` 迁移到 `jdk.internal.misc.Unsafe`，但核心 API 不变。

## 深度解析

### Unsafe 类的核心能力

```
Unsafe 能力
├── CAS 操作
│   ├── compareAndSetInt(Object, offset, expect, update)
│   ├── compareAndSetLong(Object, offset, expect, update)
│   ├── compareAndSetObject(Object, offset, expect, update)
│   └── getAndAddInt / getAndSetInt 等
│
├── 内存操作
│   ├── allocateMemory(size)       — 分配堆外内存
│   ├── freeMemory(address)        — 释放堆外内存
│   ├── putInt/getInt(address)     — 直接读写内存
│   └── allocateInstance(Class)    — 不调用构造器创建对象
│
├── 字段偏移量
│   └── objectFieldOffset(Field)   — 获取字段在对象中的偏移量
│
├── 线程调度
│   ├── park(boolean, long)        — 阻塞当前线程
│   └── unpark(Thread)             — 唤醒指定线程
│
└── 其他
    ├── shouldBeInitialized(Class) — 确保类已初始化
    ├── fullFence()                — 内存屏障
    └── loadFence() / storeFence() — 读/写屏障
```

### CAS 相关方法

```java
// JDK9+: jdk.internal.misc.Unsafe
// JDK8:   sun.misc.Unsafe

// int CAS
native boolean compareAndSetInt(Object o, long offset, int expected, int x);

// long CAS
native boolean compareAndSetLong(Object o, long offset, long expected, long x);

// Object 引用 CAS
native boolean compareAndSetObject(Object o, long offset, Object expected, Object x);

// 获取并更新（返回旧值）
native int getAndAddInt(Object o, long offset, int delta);
native int getAndSetInt(Object o, long offset, int newValue);
```

### 工作原理：字段偏移量

```java
// AtomicInteger 源码
public class AtomicInteger {
    private static final Unsafe U = Unsafe.getUnsafe();
    // 获取 value 字段在对象内存中的偏移量
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
    private volatile int value;

    public final boolean compareAndSet(int expected, int newValue) {
        // 通过偏移量直接定位到 value 字段，执行硬件 CAS
        return U.compareAndSetInt(this, VALUE, expected, newValue);
    }
}
```

内存布局示意：
```
AtomicInteger 对象
┌──────────────────────────┐
│ 对象头 (12 bytes)         │
├──────────────────────────┤
│ value  ← offset VALUE    │ ← CAS 操作的目标位置
├──────────────────────────┤
│ ...其他字段              │
└──────────────────────────┘
```

### 获取 Unsafe 实例

```java
// 方式1：反射（推荐，用户代码使用）
Unsafe unsafe = null;
Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
unsafe = (Unsafe) field.get(null);

// 方式2：JDK 内部直接调用
Unsafe unsafe = Unsafe.getUnsafe(); // 需要 Bootstrap ClassLoader 加载的类才能调用
```

## 易错点/踩坑

- ❌ 认为 Unsafe 只能用于 CAS——它还能操作堆外内存、线程调度等
- ❌ JDK9 后还能直接 `import sun.misc.Unsafe`——已迁移到 `jdk.internal.misc.Unsafe`
- ✅ 不建议在业务代码中直接使用 Unsafe——API 不稳定，且直接操作内存不安全

## 代码示例

```java
// 手动实现一个简易 AtomicReference
public class MyAtomicReference<T> {
    private static final Unsafe U;
    private static final long VALUE_OFFSET;
    private volatile T value;

    static {
        try {
            U = getUnsafe();
            VALUE_OFFSET = U.objectFieldOffset(
                MyAtomicReference.class, "value"
            );
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    public boolean compareAndSet(T expected, T newValue) {
        return U.compareAndSetObject(this, VALUE_OFFSET, expected, newValue);
    }

    public T get() { return value; }

    public void set(T newValue) { value = newValue; }

    private static Unsafe getUnsafe() throws Exception {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        return (Unsafe) f.get(null);
    }
}
```

### VarHandle：JDK9+ 的安全替代

Unsafe 虽然强大，但 API 不稳定、安全性差。JDK9 引入了 `VarHandle` 作为标准替代：

```java
// VarHandle 使用方式（JDK9+）
public class VarHandleDemo {
    private int value;

    public void demo() throws Exception {
        // 获取 VarHandle
        VarHandle VH = MethodHandles.lookup()
            .findVarHandle(VarHandleDemo.class, "value", int.class);

        // 模拟 Unsafe 的 CAS
        VH.compareAndSet(this, 10, 20);   // CAS(value, 10, 20)
        VH.getAndAdd(this, 5);            // getAndAddInt
        VH.getAndSet(this, 100);          // getAndSetInt

        // 普通读写（自动内存屏障）
        VH.set(this, 50);                 // volatile 写
        int v = (int) VH.get(this);      // volatile 读
    }
}
```

**Unsafe vs VarHandle vs AtomicInteger 对比**：

| 维度 | Unsafe | VarHandle | AtomicInteger |
|------|--------|-----------|---------------|
| JDK 版本 | 所有 | JDK9+ | 所有 |
| API 稳定性 | ❌ 不稳定 | ✅ 标准 API | ✅ 稳定 |
| 安全性 | ❌ 可越界访问 | ✅ 有边界检查 | ✅ 最安全 |
| 功能丰富度 | 最强 | 强（支持多种操作） | 专用（仅 int/long） |
| 学习成本 | 高 | 中 | 低 |
| 适用场景 | JDK 内部/框架开发 | 需要高性能字段操作 | 业务代码首选 |

**VarHandle 的优势操作**：

```java
// 原子更新（CAS 系列）
varHandle.compareAndSet(obj, expect, update);
varHandle.weakCompareAndSetPlain(obj, expect, update); // 无 volatile 语义
varHandle.getAndAdd(obj, delta);

// 内存屏障
varHandle.fullFence();    // 全屏障
varHandle.loadLoadFence(); // 读屏障
varHandle.storeStoreFence(); // 写屏障

// 数组操作
varHandle.get(array, index);           // 索引访问
varHandle.set(array, index, value);    // 索引写入
```

**VarHandle 的使用建议**：

```java
// ✅ 业务代码优先使用 AtomicInteger/AtomicReference
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();

// ⚠️ 需要自定义原子操作时使用 VarHandle
VarHandle VH = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "flag", boolean.class);
VH.compareAndSet(this, false, true);

// ❌ 不要直接使用 Unsafe
// Unsafe.getUnsafe() 仅对 Bootstrap ClassLoader 加载的类可用
// 业务代码需反射获取，存在安全风险
```

### Unsafe 的实际应用场景

尽管 Unsafe 不推荐在业务代码中使用，但它在框架层面有广泛应用：

```java
// 场景1：Netty 的 DirectByteBuffer
// ByteBuffer.allocateDirect() 底层调用 Unsafe.allocateMemory()
// 数据存储在堆外内存，零拷贝跳过 GC

// 场景2：Kryo/Fastjson 的对象序列化
// Unsafe.allocateInstance() 不调用构造器，快速创建对象

// 场景3：Disruptor 的 RingBuffer
// 无锁队列，依赖 Unsafe 的 CAS 保证线程安全

// 场景4：Guava 的 LongAdder/Striped64
// JDK 内部类直接使用 Unsafe 实现分段 CAS
```
