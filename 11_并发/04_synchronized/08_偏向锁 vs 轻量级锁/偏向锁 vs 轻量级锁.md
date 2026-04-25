---
title: 偏向锁 vs 轻量级锁
tags:
  - Java/并发
  - 对比型
module: 04_synchronized
created: 2026-04-18
---

# 偏向锁 vs 轻量级锁（无竞争用偏向，短竞争用轻量）

## 先说结论

偏向锁适用于**无竞争**场景（单线程重入），通过记录线程 ID 实现近乎零开销；轻量级锁适用于**短时间的轻度竞争**场景，通过自旋 + CAS 避免内核态切换。两者都是 synchronized 的优化，但适用场景不同。

## 深度解析

### 核心对比

| 维度         | 偏向锁                 | 轻量级锁                   |
| ------------ | ---------------------- | -------------------------- |
| 适用场景     | 几乎没有竞争            | 短时间的轻度竞争           |
| 实现机制     | Mark Word 记录线程 ID   | Lock Record + CAS          |
| 重入开销     | 零（比对线程 ID）      | 极小（检查 Lock Record）   |
| 竞争处理     | 撤销偏向锁              | 自旋等待                   |
| 线程状态     | RUNNABLE               | RUNNABLE（自旋中）         |
| JDK 版本     | JDK 6~14，JDK 15 废弃  | JDK 6+，一直可用           |
| 撤销/升级成本 | 高（safepoint 暂停）    | 中（CAS 重试 + 自旋）     |

### 选择策略

- 单线程反复获取同一把锁 → 偏向锁最优
- 两个线程交替获取 → 轻量级锁合适
- 多个线程激烈竞争 → 重量级锁

### JDK 15 之后的变化

JDK 15 废弃偏向锁后，默认锁升级路径变为：无锁 → 轻量级锁 → 重量级锁。现代 JVM 的轻量级锁性能已经足够好，偏向锁的边际收益有限。

## 易错点/踩坑

- ❌ 以为偏向锁比轻量级锁更"高级"
- ✅ 两者适用不同场景，没有绝对优劣
- ❌ 以为 JDK 15 之后偏向锁还能用
- ✅ JDK 18 正式移除偏向锁
- ❌ 以为轻量级锁没有性能开销
- ✅ CAS 操作和自旋都会消耗 CPU 资源

## 代码示例

```java
public class BiasVsLightweightDemo {
    private static final Object lock = new Object();

    // 场景1：偏向锁适用 —— 单线程重复加锁
    public void singleThread() {
        for (int i = 0; i < 1000000; i++) {
            synchronized (lock) { /* 几乎零开销 */ }
        }
    }

    // 场景2：轻量级锁适用 —— 两个线程交替竞争
    public void twoThreads() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                synchronized (lock) { /* 自旋+CAS */ }
            }
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100000; i++) {
                synchronized (lock) { /* 自旋+CAS */ }
            }
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
```

## 关联知识点

