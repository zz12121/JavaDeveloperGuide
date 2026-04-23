---
title: 反射调用方法
tags:
  - Java/反射
  - 场景型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射调用方法

## Q：如何通过反射调用方法？

**A：** 通过 `Method.invoke()` 调用：

```java
// 1. 获取 Method 对象
Method method = clazz.getMethod("methodName", ParamType.class);

// 2. 调用方法
Object result = method.invoke(target, args);  // 实例方法，第一个参数是目标对象
Object result = method.invoke(null, args);    // static 方法，第一个参数传 null
```

## Q：如何通过反射调用私有方法？

**A：** 使用 `getDeclaredMethod()` 获取私有方法，然后 `setAccessible(true)`：

```java
Method method = clazz.getDeclaredMethod("privateMethod", String.class);
method.setAccessible(true);
method.invoke(target, "参数");
```

## Q：Method.invoke() 方法执行异常如何处理？

**A：** 如果被调用的方法本身抛出异常，会被包装为 `InvocationTargetException`。真正的异常通过 `e.getCause()` 获取：

```java
try {
    method.invoke(target, args);
} catch (InvocationTargetException e) {
    Throwable realException = e.getCause();
    // 处理方法内部真正抛出的异常
}
```

## Q：反射调用方法有性能优化方案吗？

**A：** 反射查找 `Method` 对象的开销大，但调用 `invoke()` 的开销相对可控。优化方案：
1. **缓存 Method 对象**：在初始化时查找并缓存，避免重复查找
2. **JDK 7+ 的 MethodHandle**：比反射更快，接近直接调用的性能
3. **JIT 优化**：JVM 会优化频繁调用的反射方法
