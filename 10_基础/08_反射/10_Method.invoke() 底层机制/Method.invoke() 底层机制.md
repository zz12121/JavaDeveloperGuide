---
title: "Method.invoke() 底层机制"
tags:
  - Java/反射
  - 原理型
  - 底层实现
module: 08_反射
created: 2026-04-25
---

# Method.invoke() 底层机制

## 从表面到本质

```java
// 调用反射方法的标准写法
Method method = clazz.getMethod("methodName", paramTypes);
Object result = method.invoke(obj, args);
```

表面上看是普通方法调用，实际上内部经历了 ** Inflation 机制** 和 **字节码生成** 的复杂过程。

## 源码解析（JDK 7+）

### 核心源码（java.lang.reflect.Method）

```java
public Object invoke(Object obj, Object... args) throws ... {
    // 参数校验
    if (override) {
        // 如果设置了 setAccessible(true)，走这条捷径
        return MethodAccessorGenerator.generateSerializationCoder(obj, args);
    }

    // 生成调用器（懒加载，首次调用时才生成）
    return acquireMethodAccessor().invoke(obj, args);
}
```

### MethodAccessor 接口

`MethodAccessor` 是真正执行方法调用的抽象接口，有三种实现：

| 实现类 | 说明 |
|--------|------|
| `NativeMethodAccessorImpl` | 直接调用本地 native 方法，第一次快但会 Inflation |
| `DelegatingMethodAccessorImpl` | 委托给其他 MethodAccessor |
| `MethodAccessorGenerator`（动态生成） | 生成字节码类，性能更好（被 Inflation 后替代 native） |

### Inflation 机制（懒加载策略）

```
第1次调用
  └── NativeMethodAccessorImpl.invoke()
      └── 本地 native 调用（首次快，不需要类加载）

第15次调用（JDK 默认阈值）
  └── 触发 Inflation
      └── MethodAccessorGenerator 生成字节码类
          └── 新的 DelegatingMethodAccessorImpl（委托给字节码实现）
              └── 后续全部走生成的字节码类
```

**关键源码（Method.java）**：
```java
//inflationThreshold = sun.misc.Instrumentation.threshold
//默认 15，可以通过 -Dsun.reflect.inflationThreshold=N 调整
private static int inflationThreshold = 15;

private MethodAccessor acquireMethodAccessor() {
    MethodAccessor tmp = methodAccessor;
    if (tmp == null) {
        tmp = reflectionFactory.newMethodAccessor(this);
        setMethodAccessor(tmp);
    }
    return tmp;
}
```

**Inflation 源码（ReflectionFactory.java）**：
```java
public MethodAccessor newMethodAccessor(Method m) {
    // 如果开启了 inflation 且还未超过阈值，使用 native 实现
    NativeMethodAccessorImpl nativeAccessor =
        new NativeMethodAccessorImpl(m);

    // 返回一个 DelegatingMethodAccessorImpl，初始委托给 native
    DelegatingMethodAccessorImpl result =
        new DelegatingMethodAccessorImpl(nativeAccessor);

    // 设置回调：超过阈值后切换到字节码实现
    nativeAccessor.setParent(result);

    return result;
}
```

## Inflation 的原因与优化

### 为什么需要 Inflation？

| 调用方式 | 优点 | 缺点 |
|---------|------|------|
| Native 直接调用 | 首次调用快 | 每次调用都有 JNI 开销，无法被 JIT 内联 |
| 字节码生成调用 | 无 JNI 开销，可 JIT 优化，被充分内联 | 首次生成有类加载开销 |

**Inflation = 延迟到确定要频繁调用时才生成字节码**。一次性反射调用走 native，多次反射调用走优化的字节码。

### 禁用 Inflation（高并发场景）

```java
// JVM 参数禁用 Inflation
-Dsun.reflect.inflationThreshold=0

// 效果：第1次就生成字节码，跳过 native
// 注意：启动会变慢，但高并发下整体更快
```

## invoke() 的权限检查

### Accessibility 设置

```java
// 默认 false：每次调用都检查权限
method.setAccessible(false);

// 推荐（JDK 9+）：不推荐直接 setAccessible，抛出 SecurityException 风险
method.setAccessible(true);  // 警告：JDK 9+ 禁止对模块内部成员使用

// JDK 9+ 正确方式
method.setAccessible(AccessController.doPrivileged(
    new PrivilegedAction<Boolean>() {
        public Boolean run() {
            return method.canAccess(obj);
        }
    }
));
```

### 检查流程（callerSensitive 时）

```java
public Object invoke(Object obj, Object... args) throws ... {
    // 1. 参数检查
    if (args.length > 255) throw new IllegalArgumentException();

    // 2. 如果不是 override 模式且 callerSensitive，执行运行时检查
    if (!override) {
        Class<?> caller = Reflection.getCallerClass();
        // 检查 caller 是否有访问权限（模块系统检查）
        reflectionFilter.filter(caller);
    }

    // 3. 调用 MethodAccessor
    return methodAccessor.invoke(obj, args);
}
```

## 性能优化策略

### 1. 缓存 Method 对象

```java
// 每次 getMethod() 都遍历类层级结构，代价高
// 正确：缓存起来
Map<String, Method> methodCache = new ConcurrentHashMap<>();

public Object invoke(String methodName, Object target) {
    Method m = methodCache.computeIfAbsent(
        methodName,
        name -> target.getClass().getDeclaredMethod(name)
    );
    m.setAccessible(true);
    return m.invoke(target);
}
```

### 2. 避免反射调用基本类型自动装箱

```java
// 每次自动装箱有开销
method.invoke(obj, 1);  // Integer.valueOf(1)

// 推荐：使用 Constructor.newInstance 配合 IntFunction 或直接传数组
method.invoke(obj, new Object[]{ 1 });
```

### 3. MethodHandle（比反射更快）

```java
// MethodHandle 在 JDK 7+ 提供，比反射快 10-20 倍
// 因为 MethodHandle 底层使用 invokedynamic，可被 JIT 充分优化
MethodHandle mh = MethodHandles.lookup()
    .unreflect(method);
Object result = mh.invoke(obj, args);
```

### 性能对比

| 方式 | 相对性能 | JIT 优化 | 调用次数限制 |
|------|:---:|:---:|:---:|
| 直接调用 | 1x | ✅ 完全优化 | 无 |
| MethodHandle | 1.2-1.5x | ✅ 充分优化 | 无 |
| Inflation 后反射 | 2-5x | ✅ JIT 优化 | 无 |
| 首次反射（native） | 10-50x | ❌ 无法优化 | 无 |

## Spring AOP 中的反射优化

Spring 大量使用反射，采用了 **CglibAopProxy** 和 **JdkDynamicsAopProxy** 两种方式：

```java
// JdkDynamicsAopProxy 底层就是 Method.invoke()
// Spring 优化策略：生成代理类后，用 FastClass 替换反射调用

// 原始（慢）：
((UserService) proxy).save(user);  // 实际走 JdkDynamicProxy.invoke()

// Cglib 代理后（快）：
// 直接方法调用，无反射
// Cglib 生成的 FastClass 通过 index 直接调用方法
```

## 关联知识点

- [[反射创建对象]] — Constructor.newInstance 与 Inflation 机制相同
- [[反射与工厂模式和代理模式]] — Spring AOP 代理底层反射优化
