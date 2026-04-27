---
tags: [Java并发, JMM, happens-before, volatile, 内存屏障, 指令重排序]
module: 17_实际应用与场景
chapter: 09_JMM对并发编程的影响
---

# JMM对并发编程的影响

## Q1：JMM 与实际并发 Bug 有什么关系？

**A**：

大多数并发 Bug 都可以通过 JMM 原理解释：

1. **可见性问题**：线程修改共享变量后，其他线程看不到 → JMM 允许线程缓存副本不立即刷新
2. **有序性问题**：指令重排序导致非预期的执行顺序 → JMM 允许编译器和 CPU 重排序
3. **原子性问题**：i++ 不是原子操作，多线程下结果错误 → JMM 不保证复合操作的原子性

| 并发 Bug | JMM 原因 | 修复方案 |
|---------|---------|---------|
| 读到过期数据 | 线程缓存未刷新到主存 | `volatile` / `synchronized` |
| 指令重排序导致错误 | 编译器/CPU 优化重排 | `volatile`（内存屏障） |
| i++ 结果不对 | 非原子的读-改-写 | `AtomicXxx` / `synchronized` |
| DCL 单例返回未初始化对象 | 对象发布重排序 | `volatile` 修饰实例 |
| 线程停止标志不生效 | stop 标志不可见 | `volatile boolean` |

---

## Q2：怎么用 happens-before 判断线程安全？

**A**：

如果操作 A 和操作 B 之间能找到 happens-before 关系，则 A 的结果对 B 可见（线程安全）。

**完整 8 条规则**：
1. **程序顺序规则**：同线程前面的操作 hb 后面的操作
2. **volatile 规则**：volatile 写 hb 后续 volatile 读
3. **锁规则**：unlock hb 后续对同一锁的 lock
4. **线程启动规则**：`Thread.start()` hb 线程内所有操作
5. **线程终止规则**：线程所有操作 hb `Thread.join()` 返回
6. **中断规则**：`Thread.interrupt()` hb 检测到中断
7. **对象终结规则**：构造函数结束 hb `finalize()` 开始
8. **传递性**：A hb B，B hb C → A hb C

```java
// 典型例子：volatile 规则 + 传递性
volatile boolean flag = false;

// 线程A
data = loadData();    // 操作1（程序顺序 → hb 操作2）
flag = true;          // 操作2（volatile写 → hb 操作3）

// 线程B
if (flag) {           // 操作3（volatile读）
    use(data);        // 操作4（程序顺序 → hb 操作3）
}
// 传递性：操作1 hb 操作4 → 线程B 一定能看到 data 的最新值
```

---

## Q3：为什么 DCL 单例需要 volatile？

**A**：

`instance = new Singleton()` 实际分三步：
1. 分配内存空间
2. 初始化对象（执行构造函数）
3. 将引用 `instance` 指向内存地址

没有 `volatile` 时，2 和 3 可能被重排序为 **1→3→2**：

```
线程A：1 → 3（instance 非null，但构造函数未完成）→ 2
线程B：第一重检查：instance != null（此时 A 执行到步骤 3）
      直接 return instance → 拿到未初始化的对象 → NPE！
```

`volatile` 的 **StoreStore 屏障** 保证步骤 1→2 完成后才执行步骤 3；**StoreLoad 屏障** 保证写完后其他线程立即可见。

---

## Q4：什么是指令重排序？有哪些类型？

**A**：

为了提高执行效率，编译器和 CPU 会在不改变**单线程**语义的前提下调整指令顺序。

**重排序的3个层次**：

```
1. 编译器重排序
   // 原始代码
   a = 1;
   b = 2;
   c = a + b;
   
   // 编译器可能优化为
   b = 2;
   a = 1;  // a 和 b 互相独立，可以交换
   c = a + b;

2. CPU 指令级并行重排序（ILP）
   CPU 流水线，多条独立指令同时执行

3. 内存系统重排序（Store Buffer / Load Buffer）
   写操作先放 Store Buffer，可能比后面的读操作更晚提交到主存
```

**多线程问题示例**：

```java
int x = 0, y = 0;
boolean flag = false;

// 线程1
x = 42;        // 操作1
flag = true;   // 操作2（可能被重排序到操作1之前！）

// 线程2
if (flag) {
    System.out.println(x);  // 可能打印 0（看到 flag=true 但 x 还没写）
}
```

**防止重排序**：`volatile`（内存屏障）、`synchronized`（释放锁时刷新所有写入）。

---

## Q5：volatile 的可见性和有序性是如何实现的？

**A**：

**可见性实现**：
- 写 volatile 变量：JVM 插入 StoreLoad 屏障，强制将值刷新到主内存，并使其他 CPU 的缓存行失效
- 读 volatile 变量：JVM 插入 LoadLoad 屏障，强制从主内存重新读取，不使用缓存值

**有序性实现（4种内存屏障）**：

| 屏障类型 | 作用 | volatile 使用场景 |
|---------|------|-----------------|
| StoreStore | 前面的写操作完成才执行后续写 | volatile 写之前 |
| StoreLoad | 前面的写完成才执行后续读 | volatile 写之后 |
| LoadLoad | 前面的读完成才执行后续读 | volatile 读之后 |
| LoadStore | 前面的读完成才执行后续写 | volatile 读之后 |

```
volatile 写操作的屏障：
  [普通写]
  StoreStore 屏障   ← 保证前面所有写都完成
  [volatile 写]
  StoreLoad 屏障    ← 保证写完后读不会重排序到写前

volatile 读操作的屏障：
  [volatile 读]
  LoadLoad 屏障     ← 保证后续读不早于此读
  LoadStore 屏障    ← 保证后续写不早于此读
  [普通读/写]
```

---

## Q6：final 字段在 JMM 中有什么特殊保证？

**A**：

`final` 字段有**初始化安全保证**（Initialization Safety）：

```java
public class FinalFieldExample {
    final int x;         // final 字段
    int y;               // 普通字段

    FinalFieldExample() {
        x = 42;   // 对 x 的写
        y = 100;  // 对 y 的写（无保证）
    }
}
```

**JMM 对 final 的保证**：
1. 构造函数中对 `final` 字段的写，在引用被安全发布后，所有线程都能看到最终值（无需 `volatile`）
2. **禁止**将 `final` 字段的写重排序到构造函数之外

```java
// 安全发布（引用在构造函数完成后才赋值）
// 所有线程读到 obj 时，obj.x 保证是 42
FinalFieldExample obj = new FinalFieldExample();  // 此时 x=42 对所有线程可见

// 不安全发布（构造函数中 this 逸出）
class Unsafe {
    final int x;
    static Unsafe instance;
    Unsafe() {
        instance = this;  // ❌ this 逸出！此时 x 可能还未初始化
        x = 42;
    }
}
```

**实际意义**：不可变对象（所有字段 final）在安全发布后，无需任何额外同步即可安全共享。
