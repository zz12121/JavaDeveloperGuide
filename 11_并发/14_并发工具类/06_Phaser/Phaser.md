
# Phaser

## 核心结论

Phaser（分阶段同步器）是 JDK7 引入的**多阶段同步工具**，功能上是 CyclicBarrier 的增强版。支持**动态注册/注销参与者**、**多阶段推进**、**到达时回调**。

## 深度解析

### 核心机制

```
Phase 0: 3 个线程 await → 全部到达 → advance → Phase 1
Phase 1: 5 个线程 await → 全部到达 → advance → Phase 2
...（可无限推进）
```

### 与 CyclicBarrier 对比

| 特性 | CyclicBarrier | Phaser |
|------|---------------|--------|
| 参与者数量 | 固定 | 动态增减 |
| 阶段数 | 手动循环 | 自动推进 |
| 注销参与者 | 不支持 | `arriveAndDeregister()` |
| 中断处理 | BrokenBarrierException | `forceTermination()` |
| 屏障回调 | barrierAction | `onAdvance()` |
| 层级结构 | 不支持 | 支持子 Phaser |

### 关键方法

| 方法 | 说明 |
|------|------|
| `Phaser(int parties)` | 创建指定参与者数 |
| `arrive()` | 到达，不等待其他线程 |
| `arriveAndAwaitAdvance()` | 到达并等待其他线程 |
| `arriveAndDeregister()` | 到达并注销自己 |
| `register()` | 动态注册新参与者 |
| `bulkRegister(n)` | 批量注册 |
| `getPhase()` | 获取当前阶段号 |
| `forceTermination()` | 强制终止 |
| `onAdvance()` | 阶段推进回调（可覆写） |

### onAdvance 回调

```java
// 默认实现：如果没有注册的参与者，返回 true（终止）
// 可以覆写自定义终止条件
protected boolean onAdvance(int phase, int registeredParties) {
    return registeredParties == 0; // 默认
}

// 自定义：执行 10 轮后终止
protected boolean onAdvance(int phase, int registeredParties) {
    return phase >= 9;
}
```

## 代码示例

### 多阶段任务

```java
Phaser phaser = new Phaser(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        for (int phase = 0; phase < 3; phase++) {
            doWork(phase);
            phaser.arriveAndAwaitAdvance(); // 每阶段等待对齐
        }
    }).start();
}
```

### 动态参与者

```java
Phaser phaser = new Phaser(1); // 主线程先注册

// 随时可以注册新参与者
phaser.register();
new Thread(() -> {
    doWork();
    phaser.arriveAndDeregister(); // 完成后注销
}).start();

phaser.arriveAndAwaitAdvance(); // 主线程等待
```

### 自定义终止条件

```java
Phaser phaser = new Phaser(3) {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        System.out.println("Phase " + phase + " 完成，剩余 " + registeredParties + " 人");
        return phase >= 4 || registeredParties == 0;
    }
};
```

## 易错点与踩坑

### 1. bulkRegister 的陷阱：注册但不立即生效

```java
Phaser phaser = new Phaser(1);  // 初始 1 个

// ❌ bulkRegister 只是把计数加上去，不等待当前阶段
phaser.bulkRegister(5);  // 注册后总共 6 个

// ⚠️ 其他线程 arriveAndAwaitAdvance() 时会等待 6 个
// 但当前线程已经在第 0 阶段，不算在新的注册数里

// ✅ 正确做法：注册后要重新等待
phaser.bulkRegister(5);  // 注册
phaser.arriveAndAwaitAdvance();  // 等待所有 6 个到达

// ✅ 或者先到达再注册
phaser.arriveAndAwaitAdvance();  // 等待本轮完成
phaser.bulkRegister(5);  // 注册下一轮参与者
```

### 2. onAdvance 的调用时机容易搞错

```java
// ❌ 误以为 onAdvance 在所有线程到达后立即调用
Phaser phaser = new Phaser(3);

// 线程 A: arriveAndAwaitAdvance()
// 线程 B: arriveAndAwaitAdvance()
// 线程 C: arriveAndAwaitAdvance()  ← 最后一个到达

// ⚠️ 实际上：onAdvance 由最后一个到达的线程调用
// 在 onAdvance 返回 true 之前，其他线程都在等待

// ✅ 这意味着 onAdvance 会阻塞所有线程
// 如果 onAdvance 耗时很长，所有线程都会卡住
```

### 3. arriveAndAwaitAdvance 返回值的误解

```java
Phaser phaser = new Phaser(3);

// ❌ 误以为返回值是 phase 编号
int phase = phaser.arriveAndAwaitAdvance();  // 返回当前 phase 号

// ⚠️ 实际上是：返回的是到达时的 phase 号
// 如果到达时 phase = 5，返回的就是 5
// 但如果此时 phase 已经被 advance 了，返回的可能不是 5

// ✅ 正确理解：返回值用于判断是否应该继续
int nextPhase = phaser.arriveAndAwaitAdvance();
if (nextPhase < 0) {
    // phaser 已终止
    break;
}
```

### 4. 层级限制：树形结构的复杂度

```java
// ❌ 误以为可以无限层级的 Phaser
// 实际上：Phaser 内部有层级限制
// 层级数 = ceil(log2(parties))，最多支持 2^31 个参与者

Phaser root = new Phaser(1);
Phaser child1 = new Phaser(root, 0);
Phaser child2 = new Phaser(root, 0);

// ⚠️ 树形结构需要手动管理父子关系
// 如果层级过多，性能会下降

// ✅ 大部分场景不需要树形结构
// 如果参与者确实很多，考虑分治或用其他方案
```

## 关联知识点

