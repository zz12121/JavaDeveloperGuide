
# ThreadLocal与虚拟线程

## Q1：为什么 ThreadLocal 在虚拟线程中会有问题？

**A**：

ThreadLocal 的原理是每个线程对象内部维护一个 `ThreadLocalMap`，存储该线程的变量副本。平台线程通常只有几百个，内存开销可控。

但虚拟线程可以轻松创建**百万个**，如果每个虚拟线程都通过 ThreadLocal 存储数据（比如用户上下文、数据库连接），内存占用会呈线性增长：

- 100 万虚拟线程 × 1KB ThreadLocal 数据 = **1GB 内存**
- 如果存的是大对象（如连接池连接），问题更严重

这就是 **ThreadLocal 内存膨胀**问题。

---

## Q2：Scoped Values 是什么？如何替代 ThreadLocal？

**A**：

**Scoped Values（作用域值）** 是 JDK 21 预览、JDK 22+ 正式引入的特性，作为 ThreadLocal 在虚拟线程场景的替代方案。

核心区别：**Scoped Values 只存一份共享值，不复制**。

```java
// 定义
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// 绑定并使用
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // 作用域内所有代码都可以 CURRENT_USER.get()
    handleRequest();
});
// 作用域自动结束，值不可访问
```

| 对比 | ThreadLocal | Scoped Values |
|------|------------|---------------|
| 内存 | 每线程一份副本 | 共享一份 |
| 可变性 | 可变 | 不可变 |
| 生命周期 | 线程存活 | 显式作用域 |

---

## Q3：Spring 中的 ThreadLocal 在虚拟线程下有什么影响？

**A**：

Spring 框架大量使用 ThreadLocal：

- **`RequestContextHolder`**：存储当前 HTTP 请求
- **`SecurityContextHolder`**：存储认证信息
- **`TransactionSynchronizationManager`**：管理事务
- **`@Transactional`**：基于 ThreadLocal 管理数据库连接

在虚拟线程下，这些 ThreadLocal 都会导致内存膨胀。

**解决方案**：
- **Spring Boot 3.2+**：已内置虚拟线程支持（`spring.threads.virtual.enabled=true`），并逐步将内部 ThreadLocal 迁移到 Scoped Values
- **Spring Framework 6.2+**：`RequestContextHolder` 已支持 Scoped Values 传播
- 对于暂时未迁移的部分，确保 ThreadLocal 中只存储**轻量级数据**（如 ID 而非完整对象）

---

## Q4：现有项目迁移到虚拟线程时，ThreadLocal 怎么处理？

**A**：

分三步：

1. **识别所有 ThreadLocal 使用点**：全局搜索 `new ThreadLocal`、`ThreadLocal.withInitial`
2. **评估内存影响**：估算虚拟线程数量 × 每个副本大小
3. **逐步替换**：
   - 优先替换数据量大、线程数多的场景
   - JDK 22+ 使用 Scoped Values
   - 暂时无法替换的，确保 `finally { threadLocal.remove(); }`

对于第三方库（如 Hibernate、MyBatis）依赖 ThreadLocal 的场景，等待框架官方适配，暂时控制虚拟线程的创建数量。

---

## Q5：InheritableThreadLocal 在虚拟线程中还能继承父线程的值吗？

**A**：

**不能。** 这是 `InheritableThreadLocal` 在虚拟线程中最大的行为变化。

```java
// 平台线程：✅ 子线程继承父线程的 ThreadLocal 值
InheritableThreadLocal<String> tl = new InheritableThreadLocal<>();
tl.set("parent-value");

new Thread(() -> System.out.println(tl.get())).start();
// 输出: parent-value ✅

// 虚拟线程：❌ 不继承
Thread.startVirtualThread(() -> System.out.println(tl.get())).start();
// 输出: null ❌
```

**原因**：`InheritableThreadLocal` 的继承逻辑在 `Thread` 构造函数中触发（`Thread.init()`），而虚拟线程不是通过 `new Thread()` 创建的，JVM 内部直接构造 `VirtualThread` 对象，跳过了继承逻辑。

**替代方案**：使用 `Scoped Values`，作用域内的所有代码都能访问绑定值：

```java
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // 作用域内所有代码（包括被调用的方法）都可以 CURRENT_USER.get()
    handleRequest();
    processOrder();
});
// 作用域结束，值不可访问
```

---

## Q6：ThreadLocal 的 remove() 在虚拟线程中为什么更重要？

**A**：

在平台线程池中，线程会被复用，ThreadLocal 的值如果不清理会"泄漏"到下一次任务：

```java
// 平台线程池：任务1设置的 ThreadLocal，任务2可能读到脏数据
executor.submit(() -> {
    threadLocal.set("task1-data");
    // 忘记 remove()
});
executor.submit(() -> {
    System.out.println(threadLocal.get()); // 可能读到 "task1-data"（脏数据）
});
```

**在虚拟线程中问题更严重**：虚拟线程创建成本极低，通常"用完即弃"，不会像线程池那样复用。理论上不需要 `remove()`。

**但实际场景**：虚拟线程内部复用了少量载体线程，`ThreadLocal` 的值可能意外地通过载体线程泄漏。此外，如果虚拟线程长时间存活（如 HTTP 服务器的工作线程），ThreadLocal 累积也会导致内存膨胀。

**最佳实践**：
```java
// 虚拟线程中也要 finally remove
ThreadLocal<String> tl = new ThreadLocal<>();
try {
    tl.set(data);
    process();
} finally {
    tl.remove(); // 防止内存泄漏 + 数据混乱
}
```

**更优方案**：直接迁移到 `Scoped Values`，作用域结束自动清除，不需要手动 `remove()`。



