---
title: 反射的性能开销
tags:
  - Java/反射
  - 原理型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射的性能开销

## Q：反射比直接调用慢多少？

**A：** 反射比直接调用慢约 **10~100 倍**，主要开销来自：
1. 安全检查（访问权限校验）
2. 方法查找（通过名称和参数类型查找 Method）
3. 参数装箱拆箱（`invoke(Object, Object...)`）
4. 方法动态分发（无法被 JIT 充分内联优化）
实际差距取决于 JVM 版本和调用频率，现代 JVM 优化后差距在缩小。

## Q：如何优化反射性能？

**A：**
1. **缓存 Method/Field 对象**：避免每次调用都查找，在初始化时缓存
2. **setAccessible(true)**：关闭安全检查，减少一次校验开销
3. **使用 MethodHandle（JDK 7+）**：性能接近直接调用
4. **避免在循环/热路径中反射**：将反射操作移到循环外
```java
// 缓存示例
private static final Method GET_NAME = User.class.getMethod("getName");
// 后续直接使用 GET_NAME.invoke(target)
```

## Q：框架中大量使用反射，为什么性能还能接受？

**A：** 因为框架在启动阶段完成了大部分反射操作，运行时使用缓存：
- **Spring**：启动时扫描并缓存 Bean 的所有 Method/Field，运行时直接从缓存获取
- **MyBatis**：首次执行时缓存字段映射关系
- **Jackson**：`SerializerCache` 缓存类的序列化信息
此外，JVM 的 JIT 编译器会对频繁调用的反射方法进行优化（内联等），使得热路径上的反射性能接近直接调用。
