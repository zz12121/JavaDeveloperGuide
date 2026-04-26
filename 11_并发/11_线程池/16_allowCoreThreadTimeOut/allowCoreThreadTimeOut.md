
# allowCoreThreadTimeOut

## 核心结论

默认核心线程即使空闲也不会被回收。`allowCoreThreadTimeOut(true)` 允许核心线程在空闲超过 keepAliveTime 后被回收，使线程池在空闲时自动缩容至 0。

## 深度解析

### 原理

```java
// 开启后核心线程也用 poll(keepAliveTime) 而非 take()
pool.allowCoreThreadTimeOut(true);

// getTask() 中的判断
boolean timed = allowCoreThreadTimeOut || workerCountOf(ctl.get()) > corePoolSize;
```

- 默认 `timed` 判断：只有非核心线程（数量 > corePoolSize）才超时
- 开启后：所有线程都超时，空闲后全部回收

### 效果

```
默认（关闭）：
  线程数 ≥ corePoolSize（空闲时也不回收）

开启后：
  线程数可以降到 0（所有线程空闲超时后回收）
  新任务来时再重新创建线程
```

### 适用场景

- 任务量波动大，空闲期长
- 需要节省资源（CPU、内存）
- 不要求低延迟（冷启动可接受）

### 注意

- 配合 SynchronousQueue 或有界队列使用效果更好
- 开启后首次任务有冷启动延迟
- CachedThreadPool 本质就是 corePoolSize=0，天然支持核心线程超时

## 易错点与踩坑

### 1. 开启后核心线程也会被回收
```java
pool.allowCoreThreadTimeOut(true);
// 所有线程（包括核心线程）空闲超过 keepAliveTime 后都会被回收
// 线程数可以降到 0
// 但注意：设置之前已存在的核心线程也会受影响（不是只影响新线程）
```

### 2. 与 prestartCoreThread 冲突
```java
// 刚预热完的线程如果空闲，会被超时回收
// 所以这两个配置不应该同时使用
pool.prestartAllCoreThreads();
pool.allowCoreThreadTimeOut(true);
// 预热 → 空闲 → 超时回收 → 下次任务冷启动
```

### 3. CachedThreadPool 的"天然"实现
```java
// CachedThreadPool 就是 corePoolSize=0 + allowCoreThreadTimeOut 的效果
// 但机制不同：
// CachedThreadPool: core=0，所有线程都是"非核心"线程
// allowCoreThreadTimeOut: core>0，但核心线程也参与超时
```

### 4. 开启后的拒绝策略变化
```java
// 默认：线程数 >= corePoolSize，任务入队
// 开启后：线程数降到 0，新任务需要创建线程
// 如果创建速度跟不上 → 可能触发拒绝策略
// 配合 SynchronousQueue 使用更安全（不堆积任务）
```
## 关联知识点

