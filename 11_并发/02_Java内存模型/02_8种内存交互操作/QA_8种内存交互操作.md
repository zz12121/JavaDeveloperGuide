---
title: 8种内存交互操作
tags:
  - Java/并发
  - 问答
  - 原理型
module: 02_JMM
created: 2026-04-18
---

# 8种内存交互操作（lock/unlock/read/load/use/assign/store/write）

## Q1：JMM定义了哪8种内存交互操作？它们分别有什么作用？

**A**：JMM定义的8种内存交互操作：

| 操作 | 作用域 | 功能 |
|------|--------|------|
| **lock** | 主内存 | 将变量标记为线程独占状态，锁定对象监视器 |
| **unlock** | 主内存 | 释放变量的锁定状态，解锁对象监视器 |
| **read** | 主内存 | 从主内存读取变量值，传输到工作内存 |
| **load** | 工作内存 | 将read读取的值放入工作内存的变量副本 |
| **use** | 工作内存 | 将工作内存变量值传递给执行引擎使用 |
| **assign** | 工作内存 | 将执行引擎计算的值赋给工作内存变量 |
| **store** | 工作内存 | 将工作内存变量值传出，准备写入主内存 |
| **write** | 主内存 | 将store传来的值写入主内存变量 |

**记忆口诀**："锁读载用赋存写"（lock-read-load-use-assign-store-write）

## Q2：8种操作中有哪些必须成对出现？为什么read和load之间可以插入其他操作？

**A**：

**必须成对出现**的操作：
1. **read → load**：从主内存读取后必须载入工作内存
2. **store → write**：从工作内存存储后必须写入主内存

**可以插入其他操作的原因**：
- JMM允许指令重排序优化性能
- read和load之间可能插入其他变量的操作
- 但**相对顺序**必须保持：read在load之前，store在write之前

**示例**：
```
允许的顺序：read(x) → read(y) → load(x) → load(y)
不允许的顺序：load(x) → read(x)  // load必须在read之后
```

## Q3：assign操作后变量值会立即对其他线程可见吗？为什么？

**A**：**不会立即可见**。

**原因**：
1. assign只修改**工作内存**中的变量副本
2. 修改后的值必须通过 **store → write** 才能写回主内存
3. 其他线程要从主内存 **read → load** 才能看到新值

**可见性保证方式**：
- `volatile`：assign后立即store+write，且禁止指令重排序
- `synchronized`：unlock前自动执行store+write
- `final`：构造函数中写入，对象发布后立即可见

## Q4：为什么long和double的读写可能不是原子性的？

**A**：

**历史原因**：
- JMM规范允许将64位的long/double拆分为两个32位操作
- 早期32位JVM在32位系统上确实存在非原子性问题

**实际情况**：
- 现代64位JVM和主流硬件（x86、ARM64）都实现了64位原子操作
- 实际开发中基本不需要担心long/double的原子性问题
- 但规范层面仍保留了这种可能性

**代码示例**：
```java
// 理论上的非原子情况（实际很少发生）
private long value = 0;

// 线程A写入 0x00000000FFFFFFFF
value = 0x00000000FFFFFFFFL;

// 线程B可能读到 0x0000000000000000（前半部分）
// 或 0xFFFFFFFF00000000（后半部分）
// 或 0x00000000FFFFFFFF（完整值）
```

**建议**：对long/double的并发修改仍建议使用`volatile`或同步机制。

## Q5：lock和unlock操作与`synchronized`关键字有什么关系？

**A**：

**对应关系**：
- `synchronized`代码块进入 → **lock** 操作
- `synchronized`代码块退出 → **unlock** 操作
**但`synchronized`不止于此**：
1. **互斥性**：同一时刻只有一个线程能lock成功
2. **可见性**：unlock前自动将工作内存变量刷新到主内存
3. **有序性**：禁止指令重排序跨越同步边界

**内存操作序列**：
```
synchronized(obj) {
    // 1. lock(obj) - 锁定监视器
    // 2. 清空工作内存中锁定对象的变量副本
    // 3. read+load 重新从主内存读取
    
    x = 1;  // assign
    
    // 4. store+write 写回主内存
    // 5. unlock(obj) - 释放监视器
}
```

## Q6：8种内存操作在实际编程中如何使用？

**A**：程序员**不直接调用**这8种操作，而是通过关键字间接使用：

| 关键字/机制 | 涉及的内存操作 |
|------------|---------------|
| `volatile`读 | read → load → use |
| `volatile`写 | assign → store → write |
| `synchronized`进入 | lock → read → load |
| `synchronized`退出 | store → write → unlock |
| 普通变量读 | read → load → use（无同步保证） |
| 普通变量写 | assign → store → write（无同步保证） |

**JVM层面**：
- 编译器将Java代码编译为字节码
- JVM解释执行或JIT编译时插入相应的内存操作
- 内存屏障（Memory Barrier）保证操作顺序

---

---

## Q7：JSR-133 之后，8 种内存操作模型还重要吗？现代 JMM 更侧重什么？

**A**：**8 种操作模型在 JSR-133 之后已经不是 JMM 的核心描述方式**，但作为理解 JMM 底层机制的入门知识仍有价值。

**现代 JMM 更侧重于**：

| 描述维度 | 具体内容 |
|---------|---------|
| **Happens-Before 规则** | 8 条规则（程序顺序、锁、volatile、线程启动/终止/中断、final、传递性） |
| **内存屏障** | 4 种屏障（LoadLoad、StoreStore、LoadStore、StoreLoad） |
| **安全发布** | final 字段、static 初始化、synchronized、volatile 的发布语义 |
| **顺序一致性** | as-if-serial + Happens-Before 的双重保证 |

**为什么 Happen-Before 更重要**：
- 8 种操作是**实现层面的描述**，Happens-Before 是**编程层面的规则**
- 程序员只需要理解 Happens-Before，无需关心 lock/unlock 如何映射到 CPU 指令
- Happens-Before 规则更直观：8 条规则 → 满足任何一条 → 可见性得到保证

```java
// 理解8种操作的实用价值：
// 知道 lock/unlock 包含 read/write 序列
// 知道 volatile 写 = assign → store → write
// 知道 volatile 读 = read → load → use

// 但实际编程：
// 只需要理解 Happens-Before 规则 → 知道何时需要同步
// 只需要理解 volatile 的 HB 语义 → 知道它能保证可见性
```

> **面试建议**：当被问到 8 种内存操作时，可以补充"现代 JMM 以 Happens-Before 为核心，8 种操作是底层基础但面试更应掌握 Happens-Before 规则"。

## 关联知识点

