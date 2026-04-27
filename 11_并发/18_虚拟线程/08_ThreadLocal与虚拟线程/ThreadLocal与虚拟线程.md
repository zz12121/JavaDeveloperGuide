
# ThreadLocal与虚拟线程

## 核心结论

`ThreadLocal` 在虚拟线程环境下面临**严重的内存膨胀问题**。每个虚拟线程都有独立的 ThreadLocal 副本，百万级虚拟线程 × ThreadLocal 数据量 = 潜在 OOM。Java 引入 **Scoped Values（作用域值）** 作为 ThreadLocal 的替代方案，通过不可变共享值解决内存问题。

## 深度解析

### 问题本质：为什么 ThreadLocal 在虚拟线程中不合适？

```
ThreadLocal 的工作原理：
  每个线程对象内部维护一个 ThreadLocalMap
  key = ThreadLocal 实例，value = 该线程的副本值

平台线程场景：
  200 个平台线程 × 1KB/ThreadLocal = 200KB ✅ 完全没问题

虚拟线程场景：
  1,000,000 个虚拟线程 × 1KB/ThreadLocal = 1GB ❌ 内存膨胀
```

### 典型受影响的场景

| 场景 | ThreadLocal 用法 | 虚拟线程下的影响 |
|------|-----------------|-----------------|
| 请求上下文传递 | 存储用户信息、TraceId | 百万请求 = 百万副本 |
| 数据库连接 | 存储当前连接 | 每个虚拟线程独占一份 |
| 日期格式化 | 存储 SimpleDateFormat | 轻量数据，影响较小 |
| 事务管理 | 存储事务状态 | Spring 的 @Transactional 场景 |
| 安全上下文 | 存储认证信息 | SecurityContext 问题 |

### 解决方案：Scoped Values

Scoped Values 是 JDK 21 Preview / JDK 22+ 的正式特性，专为虚拟线程设计：

```java
// 定义一个 ScopedValue（类似 ThreadLocal）
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// 在某个作用域内绑定值
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // 在这个 run() 内，所有代码（包括被调用的方法）都可以读取 CURRENT_USER
    processRequest();  // 内部可访问 CURRENT_USER.get()
});

// run() 结束后，作用域自动关闭，值不可访问

// 读取值
User user = CURRENT_USER.get();
```

### ThreadLocal vs Scoped Values

| 特性 | ThreadLocal | Scoped Values |
|------|------------|---------------|
| 可变性 | 可变（`set` 修改） | **不可变**（绑定后不可改） |
| 生命周期 | 线程存活期间 | 显式作用域（代码块） |
| 内存模型 | 每线程一份副本 | **共享同一份** |
| 子线程继承 | 可配置（InheritableThreadLocal） | 作用域内自动可见 |
| 内存占用 | 线程数 × 数据量 | **数据量（仅一份）** |
| 适用虚拟线程 | ❌ 内存膨胀 | ✅ 完美适配 |

### Scoped Values 的限制

1. **不可变**：绑定后不能修改值，需要"修改"时只能重新开启作用域
2. **作用域限制**：只能在绑定作用域内访问，不能跨越异步边界
3. **JDK 版本要求**：JDK 21 Preview，JDK 22+ 正式可用

### InheritableThreadLocal 在虚拟线程中的行为（重要）

`InheritableThreadLocal`（可继承的 ThreadLocal）在平台线程中，子线程会继承父线程的 ThreadLocal 值：

```java
// 平台线程：子线程 ✓ 继承父线程值
InheritableThreadLocal<String> tl = new InheritableThreadLocal<>();
tl.set("parent-value");

new Thread(() -> System.out.println(tl.get())).start(); // 输出: parent-value ✅
```

**在虚拟线程中**：`InheritableThreadLocal` **不被继承**！

```java
// 虚拟线程：不继承父线程值
InheritableThreadLocal<String> tl = new InheritableThreadLocal<>();
tl.set("parent-value");

Thread.startVirtualThread(() -> System.out.println(tl.get())).start(); // 输出: null ❌
```

**原因**：虚拟线程不是通过 `new Thread()` 创建的，而是 JVM 内部构造的，不触发 `InheritableThreadLocal` 的继承逻辑。

**替代方案**：

| 需求 | 平台线程方案 | 虚拟线程方案 |
|------|-------------|-------------|
| 线程间传递上下文 | InheritableThreadLocal | **Scoped Values**（`where()` 开启作用域） |
| 异步回调传递 | InheritableThreadLocal + ThreadPool | **Scoped Values** 作用域内所有异步操作可见 |
| 第三方库依赖 | InheritableThreadLocal | 等待框架适配，或手动传参 |

```java
// 虚拟线程正确做法：用 Scoped Values 传递上下文
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // 作用域内所有虚拟线程可见
    processTask1(); // 内部可 CURRENT_USER.get()
    processTask2();
});
```

### 框架适配进展

- **Spring Framework 6.2+**：已支持 Scoped Values 用于请求上下文传递
- **Micrometer Tracing**：已适配虚拟线程的 TraceId 传播
- **Hibernate**：正在适配，目前仍依赖 ThreadLocal 管理数据库连接

### 迁移建议

```java
// 迁移前：ThreadLocal
private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

// 迁移后：ScopedValue
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
```

对于暂时无法迁移的场景（如依赖第三方库），可以：
1. 限制 ThreadLocal 存储的数据量（只存 ID，不存对象）
2. 使用 `ThreadLocal.withInitial()` 避免存 null
3. 确保在 finally 中 `remove()`，防止内存泄漏

## 关联知识点
