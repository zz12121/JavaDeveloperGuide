
# CyclicBarrier

## 核心结论

CyclicBarrier（循环栅栏）让一组线程互相等待，全部到达屏障点后一起继续执行。与 CountDownLatch 不同，**CyclicBarrier 可以重复使用**，且支持到达屏障时执行回调。

## 深度解析

### 核心机制

```
N 个线程各自执行
  ↓
调用 await() → 进入等待
  ↓
第 N 个线程调用 await() → 触发屏障
  ↓
执行 barrierAction（可选）
  ↓
所有线程被同时唤醒继续执行
  ↓
屏障重置，可以再次使用
```

### 关键方法

| 方法 | 说明 |
|------|------|
| `CyclicBarrier(int parties)` | 指定参与线程数 |
| `CyclicBarrier(int parties, Runnable action)` | 全部到达时执行 action |
| `await()` | 到达屏障，进入等待 |
| `await(timeout, unit)` | 带超时的等待 |
| `reset()` | 重置屏障（慎用，可能抛 BrokenBarrierException） |
| `getParties()` | 获取总参与数 |
| `isBroken()` | 屏障是否被破坏 |
| `getNumberWaiting()` | 当前等待线程数 |

### 内部实现

- **基于 ReentrantLock + Condition**（非 AQS 共享模式）
- 维护 `count`（剩余等待数）和 `parties`（总数）
- `await()` 获取锁后递减 count，count > 0 时 `trip.await()`（Condition等待），count == 0 时执行 action 并 `trip.signalAll()` 唤醒所有线程
- 用 **generation** 对象标记"代"，reset() 会创建新一代

### BrokenBarrierException

以下情况屏障会被标记为 broken，所有等待线程抛出 `BrokenBarrierException`：

- 某个线程在 await() 时被中断
- 某个线程 await() 超时
- 调用了 reset()

## 代码示例

### 基本用法

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程到达屏障，一起继续！");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            System.out.println(Thread.currentThread().getName() + " 到达屏障");
            barrier.await(); // 等待其他线程
            System.out.println(Thread.currentThread().getName() + " 继续执行");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }, "Thread-" + i).start();
}
```

### 分阶段计算

```java
// 多轮迭代计算，每轮所有线程必须完成才能进入下一轮
CyclicBarrier barrier = new CyclicBarrier(4, () -> {
    // 每轮完成后汇总
    System.out.println("第 " + round + " 轮完成");
});

for (int round = 0; round < 10; round++) {
    for (int i = 0; i < 4; i++) {
        new Thread(() -> {
            compute();
            barrier.await(); // 等待其他线程完成本轮
        }).start();
    }
}
```

## 易错点与踩坑

### 1. BrokenBarrierException 的"传染性"

```java
CyclicBarrier barrier = new CyclicBarrier(3);

// ❌ 一个线程被中断，其他线程的 await() 都会抛 BrokenBarrierException
Thread t1 = new Thread(() -> {
    try {
        barrier.await();
    } catch (BrokenBarrierException e) {
        System.out.println("t1 检测到栅栏损坏");
    }
});
Thread t2 = new Thread(() -> {
    try {
        Thread.sleep(1000);
        barrier.await();
    } catch (BrokenBarrierException e) {
        System.out.println("t2 检测到栅栏损坏");
    } catch (InterruptedException e) {
        System.out.println("t2 被中断");
    }
});
Thread t3 = new Thread(() -> {
    try {
        Thread.sleep(2000);
        barrier.await();
    } catch (BrokenBarrierException e) {
        System.out.println("t3 检测到栅栏损坏");
    } catch (InterruptedException e) {
        System.out.println("t3 被中断");
    }
});

t1.start(); t2.start(); t3.start();
Thread.sleep(500);
t1.interrupt();  // t1 被中断 → barrier 变为 broken 状态

// ⚠️ 所有其他线程再次 await 都会抛 BrokenBarrierException
// barrier.reset() 后才能继续使用
```

### 2. barrierAction 回调线程的不确定性

```java
// ❌ 回调中做并发敏感的操作要小心
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    // 这个回调由"最后一个到达"的线程执行
    // 在这个回调执行期间，其他两个线程还在等待
    // ⚠️ 如果回调抛异常，其他两个线程会永久阻塞！
    System.out.println("所有线程到达，开始汇总");
    // throw new RuntimeException("汇总失败");  // ❌ 不要这样做
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            doWork();
            barrier.await();  // 最后一个到达的线程会执行回调
        } catch (BrokenBarrierException | InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
}

// ✅ 正确做法：回调中捕获所有异常
CyclicBarrier safeBarrier = new CyclicBarrier(3, () -> {
    try {
        summarize();
    } catch (Exception e) {
        // 必须处理，否则其他线程永久阻塞
        throw new RuntimeException(e);
    }
});
```

### 3. await(timeout) 超时后的状态处理

```java
// ❌ await 超时后，没有 reset 栅栏，导致永久损坏
CyclicBarrier barrier = new CyclicBarrier(3);

try {
    boolean reached = barrier.await(5, TimeUnit.SECONDS);
    if (!reached) {
        System.out.println("超时，部分线程没到达");
        // ⚠️ barrier 可能已经 broken，没有 reset
    }
} catch (InterruptedException | TimeoutException | BrokenBarrierException e) {
    e.printStackTrace();
}

// ✅ 正确做法：超时后主动 reset
try {
    barrier.await(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    System.out.println("超时，强制重置");
    barrier.reset();  // 必须！
} catch (BrokenBarrierException e) {
    barrier.reset();  // 也需要！
}
```

### 4. parties 数量不可变，无法动态调整

```java
// ❌ CyclicBarrier 的 parties 是构造时固定的
CyclicBarrier barrier = new CyclicBarrier(3);

// 场景：玩家匹配，凑够 3 个人开局
// 问题：第 4 个玩家来了，无法加入现有栅栏
barrier.await();  // 第 3 个玩家到达，开局
// 第 4 个玩家只能等下一局

// ✅ 正确做法：使用 Phaser 动态注册
Phaser phaser = new Phaser(1);  // 初始 1 个注册者
phaser.register();  // 动态注册新参与者
phaser.arriveAndAwaitAdvance();  // 等待所有参与者
```

## 关联知识点

