---
title: JDK 18+锁策略变化
tags:
  - Java/并发
  - 原理型
module: 04_synchronized
created: 2026-04-18
---

# JDK 18+锁策略变化（偏向锁完全移除后）

## 先说结论

JDK 18 完全移除了偏向锁（Biased Locking），synchronized 的锁升级路径简化为：**无锁 → 轻量级锁 → 重量级锁**，不再经过偏向锁阶段。

## 深度解析

### 历史演进

| JDK 版本 | 状态 | 说明 |
|---------|------|------|
| JDK 1.6 | 引入 | 默认启用偏向锁，优化单线程获取锁场景 |
| JDK 15 | JEP 374 废弃 | 偏向锁被标记为废弃，但仍可使用 `-XX:+UseBiasedLocking` 启用 |
| JDK 17 | 默认关闭 | 默认 `-XX:-UseBiasedLocking`，可通过参数重新启用 |
| JDK 18 | 完全移除 | JEP 423，偏向锁相关代码从 JVM 中彻底删除 |

### 移除偏向锁的原因

1. **维护成本过高**
   - 批量重偏向（Bulk Rebiasing）和批量撤销（Bulk Revoking）机制复杂
   - 代码路径长，调试困难
   - 需要在对象头中存储偏向线程 ID、Epoch 等额外信息

2. **现代锁竞争模式变化**
   - 云计算、容器化环境下，线程频繁创建销毁
   - 短生命周期锁场景增多，偏向锁优势无法体现
   - 偏向撤销的开销在高竞争场景下成为瓶颈

3. **新技术替代**
   - Lock Elision（锁消除）和 Lock Coarsening（锁粗化）优化更成熟
   - 逃逸分析技术进步，轻量级锁开销已大幅降低
   - JOL（Java Object Layout）优化持续改进

### 移除后的锁升级路径对比

**JDK 8（偏向锁启用）**：
```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
       ↑           ↑           ↑
    同一线程    锁竞争/撤销    轻量级锁
    重入        偏向          膨胀
```

**JDK 18+（偏向锁移除）**：
```
无锁 → 轻量级锁 → 重量级锁
  ↑        ↑           ↑
对象创建  CAS竞争      锁膨胀
       或自旋失败
```

### 对现有代码的影响

- **无需修改业务代码**：synchronized 语法不变，只是内部实现简化
- **JVM 参数失效**：
  - `-XX:+UseBiasedLocking` / `-XX:-UseBiasedLocking` 不再生效
  - `-XX:BiasedLockingStartupDelay` 参数被移除
  - `-XX:+TraceBiasedLocking` / `-XX:+PrintBiasedLockingStatistics` 不可用
- **监控指标变化**：偏向锁相关 JMX 指标不再存在

## 易错点/踩坑

1. **误以为需要手动适配**：实际上大多数业务代码无需任何修改，性能可能略有提升
2. **混淆废弃和移除**：JDK 15 只是废弃（deprecated），JDK 18 才彻底移除
3. **参数混淆**：`-XX:-UseBiasedLocking` 在 JDK 15-17 是关闭偏向锁，在 JDK 18+ 会被 JVM 忽略
4. **认为锁升级更快**：移除偏向锁后，首次进入 synchronized 代码块会直接尝试轻量级锁，缺少了"假设无竞争"的优化

## 代码示例

### 验证偏向锁是否可用

```java
public class BiasedLockingTest {
    public static void main(String[] args) {
        // JDK 18+ 运行此代码，偏向锁相关参数均不可用
        System.out.println("Java Version: " + System.getProperty("java.version"));
        
        Object obj = new Object();
        synchronized (obj) {
            // 轻量级锁替代了偏向锁
            System.out.println("锁已获取");
        }
    }
}
```

### JVM 参数行为验证

```bash
# JDK 15-17：以下参数生效
java -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0 MyApp

# JDK 18+：偏向锁参数被忽略，JVM 会发出警告
java -XX:+UseBiasedLocking MyApp
# Warning: Option UseBiasedLocking was deprecated in JDK 15 and removed in JDK 18.

# JDK 18+：正确的启动方式（无偏向锁参数）
java MyApp
```

### 查看对象头（需引入 JOL）

```java
import org.openjdk.jol.info.ClassLayout;

public class ObjectHeaderDemo {
    public static void main(String[] args) {
        Object obj = new Object();
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        // JDK 18+ 输出中不再包含 "biased" 标记
    }
}
```

## 关联知识点

