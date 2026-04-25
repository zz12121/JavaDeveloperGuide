---
title: JDK 15偏向锁废弃
tags:
  - Java/并发
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# JDK 15偏向锁废弃

## 先说结论

JDK 15 通过 JEP 374 **默认禁用偏向锁**，JDK 18 通过 JEP 374 的后续工作**正式移除**偏向锁实现。原因是偏向锁在现代高并发 Java 应用中收益有限，而撤销开销高昂，维护成本不值得保留。

## 深度解析

### 时间线

| 版本     | 变化                                   |
| -------- | -------------------------------------- |
| JDK 6    | 引入偏向锁优化                          |
| JDK 15   | JEP 374：默认禁用偏向锁（仍可通过参数启用） |
| JDK 16   | 继续默认禁用                           |
| JDK 18   | 正式移除偏向锁代码及相关 JVM 参数       |

### 废弃原因

1. **撤销成本高**：偏向锁撤销需要在 safepoint 暂停所有线程（STW），批量撤销的开销更大
2. **实际收益有限**：现代 Java 应用（如 Web 服务、微服务）并发度高，纯粹的"无竞争"场景很少
3. **代码复杂度**：偏向锁相关的 bug 占据了 JVM 同步子系统中很大的比例
4. **替代方案足够好**：轻量级锁（CAS + 自旋）在短竞争场景下性能已经很好

### 影响

- 移除后 synchronized 的锁升级路径变为：**无锁 → 轻量级锁 → 重量级锁**
- `BiasedLockingStartupDelay`、`UseBiasedLocking`、`PrintBiasedLockingStatistics` 等参数全部废弃
- 对绝大多数应用**无影响**，少数依赖偏向锁的微基准测试可能有性能波动

## 易错点/踩坑

- ❌ 在 JDK 18+ 设置 `-XX:+UseBiasedLocking`
- ✅ 该参数已被移除，设置会报错
- ❌ 以为移除偏向锁后 synchronized 性能会变差
- ✅ 现代轻量级锁优化足够好，实际影响很小
- ❌ 以为所有 JVM 都移除了偏向锁
- ✅ OpenJDK 18+ 移除，其他 JVM（如某些旧版 Android JVM）可能仍保留

## 代码示例

```java
// JDK 15+ 默认锁升级路径
public class PostBiasedLockingDemo {
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // JDK 15+: 无锁 → 轻量级锁 → 重量级锁
        // 不再有偏向锁阶段
        synchronized (lock) {
            System.out.println("第一次加锁：无锁 → 轻量级锁");
        }

        Thread t = new Thread(() -> {
            synchronized (lock) {
                System.out.println("竞争：轻量级锁 → 可能膨胀");
            }
        });
        t.start(); t.join();
    }
}
```

## 关联知识点
