---
title: 泛型擦除桥接方法面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型擦除桥接方法

## Q1：什么是桥接方法（Bridge Method）？为什么需要它？

**A：**
桥接方法是编译器**自动生成**的合成方法，用于解决泛型擦除导致的多态冲突问题。

原因：子类实现了泛型接口并指定了具体类型（如 `Converter<String, Integer>`），重写方法是 `Integer convert(String)`；但泛型擦除后接口变成 `Object convert(Object)`。两者签名不匹配，编译器就生成一个桥接方法 `Object convert(Object)`，内部调用真正的 `Integer convert(String)`，从而维护多态。

---

## Q2：如何通过反射识别桥接方法？

**A：**
```java
for (Method m : SomeClass.class.getDeclaredMethods()) {
    if (m.isBridge()) {
        System.out.println("桥接方法：" + m.getName());
    }
}
```

桥接方法的 `Method.isBridge()` 返回 `true`，`Method.isSynthetic()` 也返回 `true`（由编译器合成）。

---

## Q3：桥接方法会影响程序运行吗？开发时需要关注吗？

**A：**
正常情况下**不影响**，由编译器透明处理，开发时无需关注。

需要关注的场景：
1. **反射遍历方法列表**时，要过滤掉桥接方法（`m.isBridge()` 判断），避免重复处理
2. **字节码增强框架**（如 ASM、Javassist）操作方法时，需要区分桥接方法和真实方法
3. **Spring AOP** 代理时，内部已处理桥接方法，一般无需额外处理
