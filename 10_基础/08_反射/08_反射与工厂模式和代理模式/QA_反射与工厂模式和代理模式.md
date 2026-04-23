---
title: 反射与工厂模式/代理模式
tags:
  - Java/反射
  - 场景型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射与工厂模式/代理模式

## Q：反射如何改进工厂模式？

**A：** 传统工厂模式每新增一个产品都要修改工厂类（if-else 分支），违反开闭原则。反射工厂通过 `Class.forName()` 或传入 `Class` 对象动态实例化，新增产品只需新增类，无需修改工厂：

```java
public static <T> T create(Class<T> clazz) {
    return clazz.getDeclaredConstructor().newInstance();
}
```
更灵活的做法是通过配置文件指定全限定类名，用 `Class.forName(className)` 动态加载。

## Q：JDK 动态代理的底层原理是什么？

**A：** JDK 动态代理基于**接口**实现，核心是 `Proxy.newProxyInstance()` 和 `InvocationHandler` 接口。
底层流程：
1. `Proxy.newProxyInstance()` 在运行时动态生成一个实现了指定接口的代理类字节码
2. 代理类中每个方法调用都会转发到 `InvocationHandler.invoke()`
3. `InvocationHandler.invoke()` 中通过 **`Method.invoke(target, args)`**（反射）调用真实对象的方法

## Q：JDK 动态代理和 CGLIB 代理有什么区别？

**A：**
- **JDK 动态代理**：基于接口，目标类必须实现接口，底层通过反射 `Method.invoke()` 调用
- **CGLIB 代理**：基于继承（生成目标类的子类），不需要接口，不能代理 final 类/方法，底层通过 FastClass 机制（生成 FastClass 类直接调用方法索引），性能更好
Spring AOP 中：有接口默认用 JDK 代理，无接口默认用 CGLIB。
