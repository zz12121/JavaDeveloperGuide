---
title: synchronized原理
tags:
  - Java/并发
  - 问答
  - 源码型
module: 04_synchronized
created: 2026-04-18
---

# synchronized原理（Mark Word，monitorenter/monitorexit，ObjectMonitor）

## Q1：synchronized 在 JVM 层面是如何实现的？

**A**：synchronized 有两种实现方式：

1. **同步代码块**：编译后插入 `monitorenter` 和 `monitorexit` 字节码指令。monitorenter 获取对象的 monitor，monitorexit 释放 monitor。monitorexit 会出现两次——正常路径和异常路径，确保异常时锁也能释放。

2. **同步方法**：在方法的常量池中设置 `ACC_SYNCHRONIZED` 标志。JVM 调用方法时检查该标志，隐式执行 monitorenter/monitorexit。

两者最终都是获取和释放对象的 monitor（监视器锁）。

---

## Q2：对象头的 Mark Word 存储了什么？

**A**：Mark Word 是对象头的一部分，存储了对象的运行时数据，包括：

- **锁标志位**（2bit）：01(无锁/偏向锁)、00(轻量级锁)、10(重量级锁)、11(GC标记)
- **偏向锁线程 ID**（54bit）：记录持有偏向锁的线程
- **epoch**（2bit）：偏向锁批量撤销的纪元标记
- **GC 年龄**（4bit）：对象在 Survivor 区经过的 GC 次数
- **hashCode**（31bit）：对象的哈希码（调用过 hashCode 后存储）

锁升级过程就是通过 CAS 修改 Mark Word 中的锁标志位来实现的。

---

## Q3：ObjectMonitor 的内部结构和工作原理？

**A**：ObjectMonitor 是 C++ 实现的监视器，在锁膨胀为重量级锁时使用：

- **_owner**：当前持有锁的线程
- **_EntryList**：竞争锁失败的线程阻塞队列（BLOCKED 状态）
- **_WaitSet**：调用 `Object.wait()` 的线程等待队列（WAITING 状态）
- **_recursions**：锁重入次数

**工作流程**：线程竞争锁时，如果失败就加入 `_EntryList` 阻塞；锁释放时唤醒 `_EntryList` 中的线程竞争。调用 `wait()` 的线程从 `_owner` 转移到 `_WaitSet`，调用 `notify()` 时从 `_WaitSet` 转移到 `_EntryList`。

```java
// synchronized 字节码层面 — monitorenter / monitorexit
// javap -c 查看字节码：
// synchronized (obj) { ... }
// → monitorenter     // 进入：获取对象监视器锁
// → ...               // 临界区代码
// → monitorexit      // 退出：释放锁

// 同步方法：
// public synchronized void method() { ... }
// → 方法常量池中设置 ACC_SYNCHRONIZED 标志
// → JVM 调用时自动获取/释放锁
```


## 关联知识点
