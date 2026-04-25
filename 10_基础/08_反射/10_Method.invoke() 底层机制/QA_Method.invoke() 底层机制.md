---
title: "QA_Method.invoke() 底层机制"
module: 08_反射
created: 2026-04-25
---

# 面试题：Method.invoke() 底层机制

### Q1：Method.invoke() 底层调用链路是怎样的？

**参考答案：**

Method.invoke() 的调用链路分为两个阶段：

**阶段一：获取 MethodAccessor**

```java
// 1. 检查是否已有缓存
MethodAccessor accessor = methodAccessor;

// 2. 如果为空，通过 ReflectionFactory 创建
//    首次创建返回 NativeMethodAccessorImpl + DelegatingMethodAccessorImpl
accessor = reflectionFactory.newMethodAccessor(this);

// 3. 缓存并返回
setMethodAccessor(accessor);
```

**阶段二：调用 MethodAccessor**

```java
// NativeMethodAccessorImpl.invoke() 内部：
public Object invoke(Object target, Object... args) {
    // 第1~15次：直接调用本地 native 方法
    // 第15次以后：触发 Inflation，生成字节码类，替换为 DelegatingMethodAccessorImpl
    return invoke0(target, args);  // native 方法
}
```

**完整链路图：**

```
method.invoke(obj, args)
  └── Method.acquireMethodAccessor()
        └── ReflectionFactory.newMethodAccessor()
              └── new DelegatingMethodAccessorImpl(
                    new NativeMethodAccessorImpl(method))
  └── MethodAccessor.invoke()
        └── Inflation 阈值内：NativeMethodAccessorImpl.invoke0() [JNI]
        └── 超过阈值：Inflation → 生成字节码类 → 替换调用器
```

---

### Q2：什么是 Inflation 机制？为什么需要它？

**参考答案：**

**Inflation = 懒加载的字节码生成策略**。JDK 默认在反射调用第 15 次时，用字节码生成的 MethodAccessor 替换掉 native 的 MethodAccessor。

| 调用方式 | 第1次 | 第15次后 |
|---------|:---:|:---:|
| native 反射 | ✅ 快 | ❌ JNI 开销大 |
| 字节码反射 | ❌ 冷启动慢 | ✅ JIT 充分优化 |

**为什么需要**：native JNI 调用无法被 JIT 内联优化，而字节码生成的调用可以被 JIT 优化成近似直接调用的性能。对于确定只调用 1-2 次的场景，native 更快；对于频繁调用的场景（如 Spring AOP），字节码更优。

**调整阈值**：`java -Dsun.reflect.inflationThreshold=0` 可禁用 Inflation（高并发场景用）。

---

### Q3：setAccessible(true) 有什么用？JDK 9+ 有什么变化？

**参考答案：**

`setAccessible(true)` 的作用：

1. **跳过权限检查**：绕过 Java 语言层面的 `private/protected` 访问修饰符限制
2. **模块系统访问**：`override = true` 时直接调用，跳过 `callerSensitive` 检查
3. **性能影响**：设置后，跳过 SecurityManager 检查，略微提升性能

**JDK 9+ 的变化（重要！）：**

```java
// JDK 8：直接 setAccessible(true) 即可
method.setAccessible(true);

// JDK 9+：强制 setAccessible 可能抛出 SecurityException
// 必须使用 PrivilegedAction 包装
method.setAccessible(
    AccessController.doPrivileged(
        new PrivilegedAction<Boolean>() {
            public Boolean run() {
                return method.canAccess(obj);
            }
        }
    )
);
```

原因是 JDK 9 的模块系统强化了访问控制，不在同一个模块的私有成员默认不可访问。

---

### Q4：反射调用和直接调用、性能差距有多大？

**参考答案：**

性能差距大约在 **10~50 倍**，Inflation 后缩小到 **2~5 倍**：

| 调用方式 | 相对性能 | 说明 |
|---------|:---:|------|
| 直接调用 | 1x | 完全内联，零开销 |
| Inflation 后反射 | 2~5x | 可被 JIT 优化，但仍有反射调用开销 |
| 首次反射（native） | 10~50x | JNI 开销 + 无法 JIT 内联 |
| MethodHandle.invoke() | 1.2~1.5x | invokedynamic 机制，最接近直接调用 |

**面试加分点**：MethodHandle 比反射快 10-20 倍，因为它使用 invokedynamic 指令，可以让 JIT 完全内联优化。

---

### Q5：Spring AOP 是如何用反射优化性能的？

**参考答案：**

Spring 在 AOP 中采用了多层优化：

**JdkDynamicAopProxy（基于 JDK 代理）**：
```java
// 原始反射调用（每次都走 invoke）
public Object invoke(Object proxy, Method method, Object[] args) {
    MethodInvocation invocation = new ReflectiveMethodInvocation(
        proxy, target, method, args);
    return interceptor.invoke(invocation);
}
```

**CglibAopProxy（基于 CGLIB）**：
```java
// 生成的 FastClass 通过 index 直接调用，无反射
// 生成的代理类方法：
public void save() {
    // 直接调用 MethodProxy.invokeSuper()，极快
    MethodInterceptor interceptor = this.CGLIB$CALLBACK_0;
    if (interceptor != null) {
        interceptor.intercept(this, ...);
    }
}
```

**Spring 的优化策略**：
1. **代理类预热**：应用启动时生成并缓存代理类
2. **Advice 链缓存**：不需要每次重新组装拦截器链
3. **JdkDynamicAopProxy vs CglibAopProxy 选择**：默认有接口用 JDK 代理，无接口用 CGLIB
