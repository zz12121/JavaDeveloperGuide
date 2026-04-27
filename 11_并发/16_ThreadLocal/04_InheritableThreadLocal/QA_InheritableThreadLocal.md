
# InheritableThreadLocal

## Q1：InheritableThreadLocal 和 ThreadLocal 有什么区别？

**A**：

- **ThreadLocal**：每个线程完全隔离，互不影响。数据存储在线程的 `threadLocals` 中
- **InheritableThreadLocal**：子线程自动继承父线程的值。数据存储在线程的 `inheritableThreadLocals` 中

继承发生在 `new Thread()` 时，`Thread.init()` 中调用 `createInheritedMap()` 浅拷贝父线程的 `inheritableThreadLocals`。

---

## Q2：InheritableThreadLocal 的拷贝是浅拷贝还是深拷贝？

**A**：

默认是**浅拷贝**。子线程的 Entry 的 value 引用的是父线程 value 的**同一个对象**。

如果需要深拷贝，可以重写 `childValue()` 方法：

```java
new InheritableThreadLocal<User>() {
    @Override
    protected User childValue(User parentValue) {
        return new User(parentValue.getName(), parentValue.getId());
    }
};
```

---

## Q3：InheritableThreadLocal 什么时候拷贝？拷贝了几次？

**A**：

**只在 `new Thread()` 时拷贝一次**，具体在 `Thread.init()` 方法中。

关键：线程池中线程**复用**时不会重新拷贝。所以第一次创建线程时拷贝了父线程的值，后续复用该线程执行新任务时，子线程的值仍然是第一次拷贝的旧值。

这就是 [[Card_186_InheritableThreadLocal问题]] 要解决的问题。

---

## Q4：InheritableThreadLocal 的典型使用场景是什么？

**A**：

1. **用户上下文传递**：主线程设置用户信息，子线程自动获取
2. **日志链路追踪**：MDC 中设置 traceId，子线程自动继承
3. **租户/多语言上下文**：父线程设置后，所有子线程自动感知

前提是**子线程是新建的**（不是线程池复用），否则需要用 TTL 或手动传递。

# InheritableThreadLocal问题

## Q1：InheritableThreadLocal 在线程池中使用有什么问题？

**A**：

`InheritableThreadLocal` 只在**创建子线程时**拷贝父线程的值。线程池中的线程是**复用**的，创建之后不会再重新拷贝父线程的值。

```
父线程 → set("user-A") → 提交任务 → 新建线程（拷贝 user-A）→ get() = "user-A" ✅
父线程 → set("user-B") → 提交任务 → 复用线程（不拷贝）   → get() = "user-A" ❌
```

核心问题：**线程池中线程的 `inheritableThreadLocals` 只在 Thread 构造时初始化一次**。

---

## Q2：如何解决 InheritableThreadLocal 在线程池中的值传递问题？

**A**：

三种方案：

1. **手动传递**：在提交任务时通过 `Runnable` 构造函数传递上下文，侵入性强
2. **TransmittableThreadLocal（TTL）**：阿里开源框架，通过装饰 `Runnable` 实现值的捕获与恢复，最推荐
3. **自定义包装**：封装 `TtlRunnable` 类似的装饰器逻辑

实际开发中推荐使用 TTL 框架，它是专门解决这个问题的。

---

## Q3：InheritableThreadLocal 的拷贝发生在什么时候？

**A**：

拷贝发生在 `Thread` 的构造函数中，具体在 `Thread.init()` 方法中：

```java
// Thread 构造函数 → init()
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

调用链：`new Thread()` → `init()` → `createInheritedMap()` → 拷贝父线程的 `inheritableThreadLocals`。

线程池中线程创建后放入池中，后续复用不会再次调用 `init()`，因此值不会更新。