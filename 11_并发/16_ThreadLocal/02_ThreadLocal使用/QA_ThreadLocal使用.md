
# ThreadLocal使用

## Q1：ThreadLocal 有哪些核心 API？

**A**：

```java
// 创建（推荐带初始值）
ThreadLocal<Integer> tl = ThreadLocal.withInitial(() -> 0);

// 设置值
tl.set(42);

// 获取值
Integer val = tl.get();  // 首次调用返回 initialValue() 的值

// 移除值
tl.remove();
```

三个核心方法：`set()`、`get()`、`remove()`。每个线程操作的是自己的 `ThreadLocalMap`，互不影响。

---

## Q2：withInitial() 和重写 initialValue() 有什么区别？

**A**：

功能相同，都是设置初始值。区别在于写法：

- **initialValue()**（JDK 8 之前）：需要创建匿名内部类
  ```java
  new ThreadLocal<SimpleDateFormat>() {
      @Override
      protected SimpleDateFormat initialValue() {
          return new SimpleDateFormat("yyyy-MM-dd");
      }
  };
  ```

- **withInitial()**（JDK 8+，推荐）：Lambda 简洁
  ```java
  ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
  ```

`withInitial()` 本质上是 `SuppliedThreadLocal`，内部也是调用 `initialValue()`，只是用 `Supplier` 包装。

---

## Q3：线程池中使用 ThreadLocal 需要注意什么？

**A**：

**必须在使用完毕后调用 `remove()`**，因为线程池会复用线程：

```java
pool.submit(() -> {
    try {
        CONTEXT.set(value);
        // 业务逻辑
    } finally {
        CONTEXT.remove();  // ⚠️ 必须清理
    }
});
```

如果不 remove：
1. 下一个复用该线程的任务 `get()` 会拿到残留值
2. 可能导致**内存泄漏**（value 一直占用内存）
3. 可能导致**业务数据串扰**（用户 A 的数据被用户 B 读到）


# ThreadLocal最佳实践

## Q1：使用 ThreadLocal 有哪些最佳实践？

**A**：

四条核心原则：

1. **声明为 `private static final`**：避免每次创建新实例导致 ThreadLocalMap 中积累无用的 key
2. **使用后必须 `remove()`**：防止内存泄漏和数据串扰
3. **try-finally 保护**：确保无论是否异常都能清理
4. **封装为工具类**：统一 set/get/clear 入口，降低遗漏风险

```java
private static final ThreadLocal<User> HOLDER = ThreadLocal.withInitial(() -> null);

// 使用
try {
    HOLDER.set(user);
    doSomething();
} finally {
    HOLDER.remove();
}
```

---

## Q2：为什么 ThreadLocal 要声明为 static？

**A**：

`static` 保证整个类只有一个 `ThreadLocal` 实例。如果声明为非 static：

- 每次创建对象都会 new 一个 `ThreadLocal`
- 每个 `ThreadLocal` 实例在 `ThreadLocalMap` 中对应一个 Entry
- 对象被回收后，`ThreadLocal` 实例的 key 变为 null（弱引用被 GC），但 value 残留在 Map 中

结果：ThreadLocalMap 中积累大量 key == null 的陈旧 Entry。

---

## Q3：Web 应用中如何安全管理 ThreadLocal？

**A**：

通过 Filter 或 Interceptor 在请求生命周期内管理：

```java
// Spring MVC Interceptor
@Override
public boolean preHandle(request, response, handler) {
    UserContextHolder.set(parseUser(request));
    return true;
}

@Override
public void afterCompletion(request, response, handler, ex) {
    UserContextHolder.clear();  // 请求结束必须清理
}
```

关键：**`afterCompletion` 在视图渲染完成后执行，确保任何情况下都会清理**。

---

## Q4：线程池场景下 ThreadLocal 有什么特别要注意的？

**A**：

线程池中线程是复用的，**上一次任务的 ThreadLocal 值会残留到下一次任务**。

解决方案：

1. **每次任务中 try-finally + remove()**（基本要求）
2. **装饰器模式**：包装 Runnable，自动清理，避免遗漏
3. **使用 TransmittableThreadLocal**（阿里 TTL）：自动传递和恢复值
4. **使用 Java 21 Scoped Values**（未来方案）：更安全的替代