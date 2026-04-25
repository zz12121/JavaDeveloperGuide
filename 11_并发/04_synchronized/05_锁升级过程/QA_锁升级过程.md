---
id: qa_31
title: 锁升级过程
tags:
  - Java/并发
  - 问答
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# 锁升级过程（无锁→偏向锁→轻量级锁→重量级锁）

## Q1：synchronized 的锁升级过程是怎样的？

**A**：synchronized 锁有四个级别，升级过程**单向不可逆**：

1. **无锁**（标志 01）：对象刚创建时的初始状态
2. **偏向锁**（标志 01）：首次加锁时，CAS 将线程 ID 写入 Mark Word。同一线程重入只需比对线程 ID，零开销
3. **轻量级锁**（标志 00）：其他线程竞争时，撤销偏向锁，在栈帧创建 Lock Record，CAS 更新 Mark Word 为指向 Lock Record 的指针。竞争线程通过自旋等待
4. **重量级锁**（标志 10）：自旋失败后膨胀，Mark Word 指向 ObjectMonitor，竞争线程进入 BLOCKED 状态

---

## Q2：为什么锁升级是单向不可逆的？

**A**：锁升级不可逆是为了**简化实现和避免性能抖动**。如果允许降级，需要在运行时不断判断是否应该降级，增加了复杂性。而且锁升级本身就意味着曾经有过竞争，降级后可能很快又需要升级，反而增加开销。

JVM 选择在编译时/启动时就确定策略，减少运行时的判断。

---

## Q3：偏向锁在 JDK 15 之后为什么被废弃？

**A**：偏向锁的设计初衷是优化"几乎不会竞争"的场景，但实际应用中：

1. **维护成本高**：偏向锁的撤销需要在 safepoint 进行，批量撤销的开销很大
2. **收益有限**：现代 JVM 的轻量级锁性能已经很好，偏向锁的额外优化效果有限
3. **代码复杂**：偏向锁相关的 bug 占了 JVM 同步代码的很大比例

JEP 374 在 JDK 15 默认禁用，JDK 18 正式移除。


> **代码示例：通过 JOL 观察锁升级过程中对象头的变化**

```java
// 依赖：org.openjdk.jol:jol-core
import org.openjdk.jol.vm.VM;
import org.openjdk.jol.info.ClassLayout;

Object obj = new Object();
System.out.println("无锁: " + ClassLayout.parseInstance(obj).toPrintable());

synchronized (obj) {
    System.out.println("轻量级锁: " + ClassLayout.parseInstance(obj).toPrintable());
}
// 偏向锁在 JDK 15+ 默认禁用，直接从无锁升级到轻量级锁
// Mark Word 最后 2 位：01(无锁/偏向) → 00(轻量级) → 10(重量级)
```

# 偏向锁

## Q1：什么是偏向锁？它是如何工作的？

**A**：偏向锁是 JDK 6 引入的优化，核心是**偏向第一个获取锁的线程**。

工作原理：

1. **首次加锁**：CAS 将线程 ID 写入 Mark Word，偏向标志位设为 1
2. **重入**：检查 Mark Word 中的线程 ID 是否等于当前线程，是则直接进入，零开销
3. **竞争**：其他线程竞争时，触发偏向锁撤销，升级为轻量级锁

偏向锁假设大部分锁不仅不存在多线程竞争，而且总是由**同一线程多次获取**，因此省去了每次加锁/解锁的 CAS 操作。

---

## Q2：为什么调用 hashCode() 会导致偏向锁撤销？

**A**：因为对象头 Mark Word 空间有限。无锁状态下 Mark Word 存储 hashCode（31bit），偏向锁状态下 Mark Word 存储 thread ID（54bit），两者不能共存。

一旦调用了 `Object.hashCode()`，hashCode 被写入 Mark Word，就无法再存储线程 ID，偏向锁立即被撤销，对象升级为无锁状态（如果需要同步则直接进入轻量级锁）。

---

## Q3：偏向锁的延迟启动是怎么回事？

**A**：JVM 启动时默认有 **4 秒的偏向锁延迟**（`-XX:BiasedLockingStartupDelay=4000`）。原因是在 JVM 启动阶段，大量类初始化和系统线程竞争锁，如果此时启用偏向锁会导致频繁的偏向撤销，得不偿失。

## 4 秒后偏向锁机制才正式生效，此时大部分系统初始化已完成，偏向锁能发挥其优化效果。

```java
// 偏向锁：单线程重入零开销
Object lock = new Object();
// 首次加锁：CAS 将线程 ID 写入 Mark Word
synchronized (lock) {
    // 重入：只需比对线程 ID，无需 CAS
    synchronized (lock) {
        // 再次重入：仍然零开销
    }
}
// 偏向锁重入开销为零，轻量级锁每次重入需要检查 Lock Record
```

# 偏向锁

## Q1：什么是偏向锁？它是如何工作的？

**A**：偏向锁是 JDK 6 引入的优化，核心是**偏向第一个获取锁的线程**。

工作原理：

1. **首次加锁**：CAS 将线程 ID 写入 Mark Word，偏向标志位设为 1
2. **重入**：检查 Mark Word 中的线程 ID 是否等于当前线程，是则直接进入，零开销
3. **竞争**：其他线程竞争时，触发偏向锁撤销，升级为轻量级锁

偏向锁假设大部分锁不仅不存在多线程竞争，而且总是由**同一线程多次获取**，因此省去了每次加锁/解锁的 CAS 操作。

---

## Q2：为什么调用 hashCode() 会导致偏向锁撤销？

**A**：因为对象头 Mark Word 空间有限。无锁状态下 Mark Word 存储 hashCode（31bit），偏向锁状态下 Mark Word 存储 thread ID（54bit），两者不能共存。

一旦调用了 `Object.hashCode()`，hashCode 被写入 Mark Word，就无法再存储线程 ID，偏向锁立即被撤销，对象升级为无锁状态（如果需要同步则直接进入轻量级锁）。

---

## Q3：偏向锁的延迟启动是怎么回事？

**A**：JVM 启动时默认有 **4 秒的偏向锁延迟**（`-XX:BiasedLockingStartupDelay=4000`）。原因是在 JVM 启动阶段，大量类初始化和系统线程竞争锁，如果此时启用偏向锁会导致频繁的偏向撤销，得不偿失。

## 4 秒后偏向锁机制才正式生效，此时大部分系统初始化已完成，偏向锁能发挥其优化效果。

```java
// 偏向锁：单线程重入零开销
Object lock = new Object();
// 首次加锁：CAS 将线程 ID 写入 Mark Word
synchronized (lock) {
    // 重入：只需比对线程 ID，无需 CAS
    synchronized (lock) {
        // 再次重入：仍然零开销
    }
}
// 偏向锁重入开销为零，轻量级锁每次重入需要检查 Lock Record
```

# 偏向锁撤销

## Q1：偏向锁撤销的过程是怎样的？为什么开销大？

**A**：偏向锁撤销必须在 **safepoint** 进行（暂停所有应用线程），过程如下：

1. 暂停持有偏向锁的线程
2. 遍历其栈帧，检查是否存在 Lock Record（锁重入记录）
3. 将 Mark Word 恢复为无锁状态
4. 升级为轻量级锁（CAS 替换为 Lock Record 指针）
5. 唤醒暂停的线程

开销大的原因：safepoint 本身就需要暂停所有线程（STW），加上遍历栈帧的操作，在高并发场景下频繁撤销会严重影响性能。

---

## Q2：什么是批量重偏向和批量撤销？

**A**：JVM 用两个阈值来分摊偏向锁撤销的成本：

- **批量重偏向（阈值 20）**：某类的偏向锁撤销达到 20 次时，将该类的 epoch 值加 1。对象下次检查时发现 epoch 不匹配，会尝试重新偏向于当前竞争线程，而不是直接升级为轻量级锁
    
- **批量撤销（阈值 40）**：撤销继续增加到 40 次时，直接禁用该类的偏向锁，所有该类对象升级为轻量级锁
    

---

## Q3：哪些操作会导致偏向锁撤销？

**A**：以下操作会触发偏向锁撤销：

1. **其他线程竞争**：非偏向线程尝试获取偏向锁
2. **调用 hashCode()**：Mark Word 空间不足，需要存 hashCode（31bit），与线程 ID（54bit）冲突
3. **调用 wait()/notify()**：需要 ObjectMonitor，必须升级为重量级锁

> **代码示例：偏向锁撤销触发条件演示**

```java
// 场景1：调用 hashCode() 触发偏向锁撤销
Object obj = new Object();
synchronized (obj) { /* 偏向锁 */ }
obj.hashCode(); // 偏向锁撤销！Mark Word 需要存储 hashCode，与线程 ID 冲突

// 场景2：其他线程竞争触发偏向锁撤销
synchronized (obj) {
    // 线程A持有偏向锁
}
new Thread(() -> {
    synchronized (obj) {
        // 线程B竞争 → 偏向锁撤销 → 升级为轻量级锁
    }
}).start();

// 场景3：wait/notify 触发膨胀为重量级锁
synchronized (obj) {
    obj.wait(); // 需要 ObjectMonitor → 直接升级为重量级锁
}
```

# 轻量级锁

## Q1：轻量级锁的加锁和解锁过程是怎样的？

**A**：

**加锁**：

1. 在当前栈帧创建 Lock Record，包含对象引用和 Displaced Mark Word
2. CAS 将对象 Mark Word 替换为指向 Lock Record 的指针（标志位改为 00）
3. CAS 成功 → 获取锁；CAS 失败 → 说明有竞争，进入自旋

**解锁**：

1. CAS 将 Displaced Mark Word 替换回对象头
2. CAS 成功 → 释放完成；CAS 失败 → 说明锁已膨胀为重量级锁，唤醒 EntryList 中的线程

---

## Q2：什么是自适应自旋？

**A**：自适应自旋是 JDK 6 引入的优化，JVM 根据历史情况动态调整自旋次数：

- 上次自旋**成功**了 → JVM 认为这次也大概率成功 → **增加**自旋次数
- 上次自旋**失败**了 → JVM 认为再自旋也是浪费 → **减少**或**跳过**自旋

相比固定自旋次数（`-XX:PreBlockSpin=10`），自适应自旋能更好地适应不同的竞争情况。

---

## Q3：轻量级锁在什么场景下会膨胀为重量级锁？

**A**：以下情况轻量级锁会膨胀为重量级锁：

1. **自旋超过阈值**：竞争线程自旋了一定次数仍然没有获取到锁
2. **竞争线程超过 2 个**：多个线程同时竞争，自旋效率低
3. **持有锁的线程调用了 wait()**：需要 ObjectMonitor 支持

膨胀过程：创建/获取 ObjectMonitor，将 Mark Word 替换为指向 ObjectMonitor 的指针（标志位 10），竞争线程进入 _EntryList 阻塞。

```java
// 轻量级锁：CAS + Lock Record
Object lock = new Object();
// 竞争时：在当前线程栈帧创建 Lock Record
synchronized (lock) {
    // CAS 将 Mark Word 替换为指向 Lock Record 的指针
    // 成功：获取锁
    // 失败：自旋等待
}
// 解锁：CAS 将 Mark Word 恢复

// 轻量级锁适用于短时间轻度竞争
// 优势：避免 OS 上下文切换（纯用户态 CAS）
```

# 轻量级锁膨胀

## Q1：轻量级锁在什么条件下膨胀为重量级锁？

**A**：以下情况会触发膨胀：

1. **自旋超过阈值**：竞争线程自旋一定次数后仍未能获取锁
2. **竞争线程 ≥ 2**：多个线程同时竞争，自旋效率极低
3. **调用 wait()/notify()**：需要 ObjectMonitor 的 _WaitSet 支持

膨胀后锁标志位从 00（轻量级锁）变为 10（重量级锁），Mark Word 指向 ObjectMonitor。

---

## Q2：轻量级锁膨胀的详细过程？

**A**：

1. 为对象分配 ObjectMonitor（或从全局空闲列表获取）
2. 将持有轻量级锁的线程设置为目标 ObjectMonitor 的 _owner
3. CAS 将对象 Mark Word 替换为指向 ObjectMonitor 的指针
4. 将自旋等待的竞争线程从"自旋状态"移入 ObjectMonitor 的 _EntryList
5. 竞争线程状态从 RUNNABLE 变为 BLOCKED，交由操作系统调度

膨胀是不可逆的，一旦变为重量级锁，即使后续没有竞争也保持重量级锁状态。

---

## Q3：为什么不能一直自旋而不膨胀？

**A**：自旋虽然避免了线程上下文切换的开销，但在以下情况下反而更差：

- **锁持有时间长**：自旋线程白白消耗 CPU
- **竞争线程多**：多个线程同时自旋，CPU 利用率暴增但实际进展缓慢
- **单核 CPU**：自旋线程占着 CPU，持有锁的线程反而得不到执行

因此 JVM 会在自旋无效时及时膨胀为重量级锁，让出 CPU 给其他任务。

> **代码示例：轻量级锁膨胀条件演示**

```java
Object lock = new Object();
int threads = 10;
CountDownLatch latch = new CountDownLatch(threads);

// 多线程竞争 → 超过自旋阈值 → 膨胀为重量级锁
for (int i = 0; i < threads; i++) {
    new Thread(() -> {
        synchronized (lock) {
            try { Thread.sleep(1); } catch (e) {}
            // 持有锁时间较长，自旋无效 → 膨胀
        }
        latch.countDown();
    }).start();
}
latch.await();
System.out.println(ClassLayout.parseInstance(lock).toPrintable());
// Mark Word 末两位为 10 → 重量级锁
```

# 重量级锁

## Q1：ObjectMonitor 的核心字段和工作原理？

**A**：ObjectMonitor 是 C++ 实现的监视器对象，核心字段：

- **_owner**：当前持有锁的线程引用
- **_EntryList**：竞争锁失败的线程双向链表（BLOCKED 状态）
- **_WaitSet**：调用 `wait()` 的线程等待集合（WAITING 状态）
- **_recursions**：锁的重入次数

**工作原理**：获取锁时检查 _owner，空闲则设为当前线程；占用则加入 _EntryList 阻塞。调用 wait() 从 _owner 转移到 _WaitSet。notify() 从 _WaitSet 转移到 _EntryList。锁释放时唤醒 _EntryList 中的线程竞争。

---

## Q2：重量级锁为什么开销大？

**A**：重量级锁的开销主要来自：

1. **线程上下文切换**：竞争失败的线程从用户态切换到内核态（BLOCKED），涉及 CPU 寄存器保存/恢复
2. **操作系统调度**：BLOCKED 线程需要 OS 调度器重新唤醒，延迟不确定
3. **ObjectMonitor 维护**：需要维护 _EntryList 和 _WaitSet 两个队列

相比之下，轻量级锁的自旋完全在用户态进行，避免了系统调用。

---

## Q3：wait()/notify() 与 ObjectMonitor 的关系？

**A**：

- `wait()`：当前线程从 _owner 移除，加入 _WaitSet，释放锁，线程状态变为 WAITING
- `notify()`：从 _WaitSet 中取一个线程移到 _EntryList，线程状态变为 BLOCKED（需要重新竞争锁）
- `notifyAll()`：将 _WaitSet 中所有线程移到 _EntryList

## 注意：被 notify 的线程并不是直接获取锁，而是回到 _EntryList 重新竞争锁。

```java
// ObjectMonitor 工作示意
Object lock = new Object();

// 线程A：获取锁成功 → _owner = 线程A
synchronized (lock) {
    lock.wait();  // 线程A：_owner=null, 线程A → _WaitSet
    // 被 notify() 唤醒后：_WaitSet → _EntryList → BLOCKED（等待锁）
}

// 线程B：竞争锁失败 → BLOCKED（进入 _EntryList）
synchronized (lock) {
    // 当线程A释放锁后，线程B被唤醒获取锁
}

// wait/notify 与 ObjectMonitor 的关系：
// wait(): _owner → _WaitSet（释放锁）
// notify(): _WaitSet → _EntryList（需要重新竞争锁）
```

## 关联知识点

