---
title: CAS的问题
tags:
  - Java/并发
  - 原理型
module: 05_CAS
created: 2026-04-18
---

# CAS的问题（ABA问题/循环时间长/只能保证一个变量原子性）

## 先说结论

CAS 存在三个经典问题：**ABA 问题**（值被改了又改回来，CAS 误判为未变）、**自旋开销**（长时间竞争导致 CPU 空转）、**只能保证单个变量的原子操作**（无法同时原子更新多个变量）。

## 深度解析

### 问题一：ABA 问题

```
时间线：
T1: 线程1 读 V=10 (A=10)
T2: 线程2 CAS(V,10,11) ✓ → V=11
T3: 线程3 CAS(V,11,10) ✓ → V=10
T4: 线程1 CAS(V,10,20) ✓ → V=20  ⚠️ 看起来没变，实际已经被改过！
```

**危害**：链表/栈等数据结构中，节点被删除又插入同值节点，CAS 无法感知，可能导致数据结构损坏。

**解决方案**：
- `AtomicStampedReference`：使用版本号（stamp），每次修改版本号+1
- `AtomicMarkableReference`：使用布尔标记位（true/false）

### 问题二：自旋时间长 / 开销大

```
高竞争场景：
线程1: CAS 失败 → 重试 → 失败 → 重试 → 失败 → ...（空转浪费 CPU）
线程2: CAS 失败 → 重试 → 失败 → 重试 → 失败 → ...
线程3: CAS 失败 → 重试 → 失败 → 重试 → 失败 → ...
```

**解决方案**：
- 限制自旋次数（JVM 自适应自旋 `-XX:+UseSpinning`）
- 使用 `LongAdder` 分段 CAS 降低竞争
- 竞争激烈时退化为阻塞锁（如锁升级）

### 问题三：只能保证一个变量的原子操作

```java
// ❌ 无法同时原子更新两个变量
int x = 0, y = 0;
// CAS(x, 0, 1) + CAS(y, 0, 1) 不是原子的！

// ✅ 解决方案
// 方案1：封装为对象，使用 AtomicReference
class Pair {
    final int x, y;
}
AtomicReference<Pair> ref = new AtomicReference<>(new Pair(0, 0));

// 方案2：使用锁
// 方案3：JDK16+ 使用 VarHandle
```

## 易错点/踩坑

- ❌ 认为 ABA 问题在所有场景都有害——简单计数器场景 ABA 无影响
- ❌ CAS 失败就直接放弃——通常配合自旋重试
- ✅ ABA 问题的严重程度取决于业务场景，链表/栈等指针操作场景最危险

## 代码示例

```java
// ABA 问题演示
public class ABAProblem {
    private static AtomicInteger value = new AtomicInteger(10);

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            int a = value.get();          // 读到 10
            try { Thread.sleep(1000); }   // 模拟耗时操作
            boolean success = value.compareAndSet(a, 20); // CAS(10, 20) — 可能成功！
            System.out.println("Thread1 CAS result: " + success); // true
        });

        Thread t2 = new Thread(() -> {
            value.compareAndSet(10, 11);  // 10 → 11
            System.out.println("10 → 11");
            value.compareAndSet(11, 10);  // 11 → 10
            System.out.println("11 → 10, ABA 发生！");
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

# ABA问题

## 先说结论

ABA 问题是 CAS 的经典缺陷：变量值从 A→B→A 后，CAS 仍会成功。JDK 提供两种解决方案——`AtomicStampedReference`（版本号机制）和 `AtomicMarkableReference`（标记位机制），通过在 CAS 比较时额外校验版本/标记来解决。

## 深度解析

### ABA 问题详解

```
              时间线
              ─────────────────────────────────────────►

线程1:  读 A=10 ──────────────── CAS(10,20) ✓ (误判！)
                              │
线程2:              CAS(10,11) → CAS(11,10)
                     A→B         B→A
```

线程1 看到的值虽然仍是 10，但中间已经被线程2 修改过了。

### 危害场景：链表节点删除

```
栈结构（头插法）：
初始:  HEAD → A → B → C

线程1: 弹出 A，记录 next=B
线程2: 弹出 A，弹出 B，再压入 A（A 被回收后重新分配）
        HEAD → A' → C    （A' 和 A 地址不同但值相同）

线程1: CAS(HEAD, A, B) → A 已不在栈顶，但 A' 的值等于 A
       实际需要 A' != A，CAS 无法区分！
```

### 解决方案一：AtomicStampedReference

```java
// 构造：初始值 + 初始版本号
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

// CAS 时同时比较值和版本号
int stamp = ref.getStamp();  // 获取当前版本号
ref.compareAndSet(10, 20, stamp, stamp + 1);
//          期望值 新值 期望版本  新版本
```

每次修改值时版本号自增，即使值回到 10，版本号也变了：

```
初始:     value=10, stamp=0
线程2:    CAS(10,11,0,1) → value=11, stamp=1
线程2:    CAS(11,10,1,2) → value=10, stamp=2
线程1:    CAS(10,20,0,1) → stamp 不匹配(0≠2) → 失败！✅
```

### 解决方案二：AtomicMarkableReference

```java
// 构造：初始值 + 初始标记（boolean）
AtomicMarkableReference<Integer> ref = new AtomicMarkableReference<>(10, false);

// CAS 时同时比较值和标记
boolean mark = ref.isMarked();
ref.compareAndSet(10, 20, false, true);
//          期望值 新值  期望标记  新标记
```

适用场景：不需要记录修改次数，只需标记"是否被修改过"（如垃圾回收标记）。

### 两种方案对比

|维度|AtomicStampedReference|AtomicMarkableReference|
|---|---|---|
|版本控制|int 版本号（可多次递增）|boolean 标记（只有 true/false）|
|使用场景|需要知道被改了几次|只需知道"是否被改过"|
|ABA 检测|精确检测每一次修改|只检测是否被修改过|
|典型应用|链表/栈操作|垃圾回收标记|

## 易错点/踩坑

- ❌ 认为 ABA 问题只在理论上存在——在无锁数据结构（栈、队列、链表）中确实会导致数据损坏
- ❌ AtomicStampedReference 的版本号会自动递增——需要手动传入新版本号
- ✅ 简单计数器场景 ABA 无影响，可以不用版本号

## 代码示例

```java
public class ABASolution {
    public static void main(String[] args) throws Exception {
        AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(10, 0);

        Thread t1 = new Thread(() -> {
            int stamp = ref.getStamp();
            System.out.println("t1 读到值=" + ref.getReference() + ", 版本=" + stamp);
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            boolean ok = ref.compareAndSet(10, 20, stamp, stamp + 1);
            System.out.println("t1 CAS: " + ok); // false，版本号不匹配
        });

        Thread t2 = new Thread(() -> {
            int stamp = ref.getStamp();
            ref.compareAndSet(10, 11, stamp, stamp + 1); // 版本 0→1
            System.out.println("t2: 10→11, 版本" + ref.getStamp());
            stamp = ref.getStamp();
            ref.compareAndSet(11, 10, stamp, stamp + 1); // 版本 1→2
            System.out.println("t2: 11→10, 版本" + ref.getStamp());
        });

        t1.start(); t2.start();
        t1.join();  t2.join();
        // 输出：t1 CAS: false — 成功阻止 ABA 问题
    }
}
```


# 自旋CAS

## 先说结论

自旋 CAS 是指 CAS 失败后不放弃，而是在循环中不断重试直到成功。它适用于**竞争不激烈、临界区短**的场景。竞争激烈时自旋会浪费 CPU 资源，JVM 通过自适应自旋和锁升级机制来优化。

## 深度解析

### 自旋 CAS 基本模式

```java
// 经典自旋 CAS 模式
do {
    oldValue = read();      // 1. 读当前值
    newValue = compute(oldValue); // 2. 计算新值
} while (!CAS(oldValue, newValue)); // 3. CAS 失败则重试
```

### 自旋的优缺点

```
优点 ✅
├── 无线程阻塞/唤醒开销（无内核态切换）
├── 用户态完成，响应速度快
└── 适用于短临界区（纳秒~微秒级）

缺点 ❌
├── 竞争激烈时 CPU 空转，浪费资源
├── 无法保证公平性（先到不一定先得）
└── N 个线程竞争，理论最坏 O(N) 次重试
```

### JVM 自适应自旋

JDK6 引入自适应自旋，JVM 根据运行情况动态调整自旋次数：

```
第一次自旋失败 → JVM 认为可能还会成功 → 继续自旋
连续多次失败   → JVM 认为竞争激烈   → 放弃自旋，升级为重量级锁

相关参数：
-XX:PreBlockSpin=10      （JDK5：固定自旋次数，默认10）
-XX:+UseSpinning         （JDK6+：启用自适应自旋，默认开启）
```

### 适用场景判断

```
并发度        推荐方案
────────────────────────────
1~2 线程     自旋 CAS ✅（最佳）
3~10 线程    自旋 CAS ⚠️（可接受）
10+ 线程     分段 CAS / 阻塞锁 ✅
极端高并发    LongAdder / 阻塞锁 ✅
```

## 易错点/踩坑

- ❌ 自旋次数越多越好——自旋是 CPU 空转，应该尽快让出 CPU
- ❌ 所有场景都用自旋 CAS——临界区大（如 IO 操作）会导致长时间自旋
- ✅ 自旋前可先 `Thread.yield()` 让出时间片，减少无效竞争

## 代码示例

```java
// 自旋 CAS 实现非阻塞计数器
public class SpinCASCounter {
    private volatile int count;

    public void increment() {
        int oldValue;
        do {
            oldValue = count;
        } while (!compareAndSwap(oldValue, oldValue + 1));
    }

    // 模拟 CAS（实际用 Unsafe）
    private synchronized boolean compareAndSwap(int expected, int newValue) {
        if (count == expected) {
            count = newValue;
            return true;
        }
        return false;
    }

    public int get() { return count; }
}

// 优化版：加入 yield 减少空转
public void incrementWithYield() {
    int retries = 0;
    int oldValue;
    do {
        oldValue = count;
        if (++retries > 100) {
            Thread.yield(); // 自旋次数过多，主动让出 CPU
            retries = 0;
        }
    } while (!compareAndSwap(oldValue, oldValue + 1));
}
```

## 关联知识点
