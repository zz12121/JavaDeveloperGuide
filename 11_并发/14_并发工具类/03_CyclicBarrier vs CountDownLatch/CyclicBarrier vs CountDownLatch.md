
# CyclicBarrier vs CountDownLatch

## 核心结论

两者都是线程同步工具，但**关注点不同**：CountDownLatch 是"主线程等 N 个事件完成"，CyclicBarrier 是"N 个线程互相等待到达同一时间点"。CountDownLatch 一次性不可重用，CyclicBarrier 可重复使用。

## 深度解析

### 全面对比

| 对比维度 | CountDownLatch | CyclicBarrier |
|---------|----------------|---------------|
| **等待角色** | 主线程等待（或1个线程等N个） | N个线程互相等待 |
| **计数方向** | 递减（N → 0） | 递减（N → 0），然后重置 |
| **可重用** | ❌ 一次性 | ✅ 可循环使用 |
| **回调** | 无 | 支持 barrierAction |
| **底层实现** | AQS 共享模式 | ReentrantLock + Condition |
| **重置** | 不支持 | `reset()` |
| **异常处理** | 不涉及 | BrokenBarrierException |
| **使用场景** | 等待多个任务完成 | 多线程互相等待对齐 |

### 核心区别图解

```
CountDownLatch:
  主线程 ──await()──> 阻塞等待
  Worker1 ──countDown()──┐
  Worker2 ──countDown()──┼──> count=0 → 唤醒主线程
  Worker3 ──countDown()──┘

CyclicBarrier:
  Thread1 ──await()──┐
  Thread2 ──await()──┼──> 全部到达 → barrierAction → 全部继续
  Thread3 ──await()──┘
```

### 使用场景选择

| 场景 | 选择 | 原因 |
|------|------|------|
| 主线程等待 N 个子任务 | CountDownLatch | 单向等待，一次性 |
| 多线程迭代计算 | CyclicBarrier | 需要重复使用 |
| 应用启动等待初始化 | CountDownLatch | 一次性等待 |
| 并发测试多轮同步 | CyclicBarrier | 每轮重新对齐 |
| 并行数据分片处理 | 两者均可 | CountDownLatch 更简洁 |

### 互相模拟

用 CountDownLatch 模拟 CyclicBarrier 很难（不可重用），但用 CyclicBarrier 模拟 CountDownLatch 也不方便（需要创建 N-1 个工作线程 + 1 个"收集线程"）。

**选择原则**：
- 主线程等结果 → CountDownLatch
- 线程间互相等待 → CyclicBarrier

## 易错点与踩坑

### 1. 混淆使用场景

```java
// ❌ 用 CountDownLatch 实现多线程互相等待（错误）
CountDownLatch latch = new CountDownLatch(3);
new Thread(() -> { latch.await(); doWork(); }).start();
new Thread(() -> { latch.await(); doWork(); }).start();
new Thread(() -> { latch.countDown(); }).start();  // 只触发一次

// ⚠️ 问题：CountDownLatch 一次性，3 个线程都 await 会永久阻塞

// ✅ 正确做法：使用 CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3);
new Thread(() -> { barrier.await(); doWork(); }).start();
new Thread(() -> { barrier.await(); doWork(); }).start();
new Thread(() -> { barrier.await(); doWork(); }).start();
```

### 2. 选型错误导致代码复杂

```java
// ❌ 场景：多阶段任务处理
// 第一阶段：并行计算
// 第二阶段：并行计算（依赖第一阶段结果）
// 第三阶段：汇总

// ❌ 错误用 CountDownLatch（需要重新创建）
CountDownLatch latch1 = new CountDownLatch(N);
latch1.await();  // 等待第一阶段
CountDownLatch latch2 = new CountDownLatch(N);  // 需要重新创建！
latch2.await();  // 等待第二阶段

// ✅ 正确用 Phaser（天然支持多阶段）
Phaser phaser = new Phaser(N);
for (int phase = 0; phase < 3; phase++) {
    phaser.arriveAndAwaitAdvance();  // 自动进入下一阶段
}
```

### 3. 忽略异常处理差异

```java
// ❌ CyclicBarrier 一个线程异常，其他线程永久阻塞
CyclicBarrier barrier = new CyclicBarrier(3);
try {
    barrier.await();  // 如果其他线程抛异常，这里会收到 BrokenBarrierException
} catch (BrokenBarrierException e) {
    // 需要处理栅栏损坏
    barrier.reset();  // 重置才能继续
}

// ❌ CountDownLatch 不需要处理这种异常
CountDownLatch latch = new CountDownLatch(3);
latch.await();  // 线程异常不会影响 latch

// ✅ 选择时要考虑异常处理需求
// 需要精细异常控制 → 选具体工具
// 不关心异常 → CountDownLatch 简单
```

### 4. 性能差异被忽略

```java
// ❌ 误以为 CyclicBarrier 和 CountDownLatch 性能一样
// 实际上：
// - CountDownLatch: AQS 共享模式，唤醒所有线程 O(n)
// - CyclicBarrier: ReentrantLock + Condition，唤醒所有线程 O(n)

// ⚠️ 但 CyclicBarrier 有额外的：
// 1. barrierAction 执行（单线程）
// 2. 重置操作

// ✅ 高并发场景：优先用 CountDownLatch（更简单）
// ✅ 需要循环同步：只能用 CyclicBarrier
```

## 关联知识点
