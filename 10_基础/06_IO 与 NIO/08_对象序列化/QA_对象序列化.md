---
title: 对象序列化
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# 对象序列化

## Q1：什么是 Java 序列化？

**A：**
将 Java 对象转换为字节序列（序列化），以便存储到文件或在网络中传输；从字节序列恢复为 Java 对象（反序列化）。

---

## Q2：Serializable 和 Externalizable 的区别？

**A：**

| 维度 | Serializable | Externalizable |
|------|-------------|----------------|
| 方法 | 无（标记接口） | writeExternal / readExternal |
| 控制方式 | JVM 自动序列化所有非 transient 字段 | 开发者手动控制 |
| 性能 | 较低 | 更高 |
| 无参构造器 | 不要求 | 必须 |

---

## Q3：serialVersionUID 有什么作用？

**A：**
用于版本校验。反序列化时比较字节流中的 UID 和类的 UID，不一致则抛 `InvalidClassException`。不定义时 JVM 会自动生成，但类修改后 UID 可能变化导致反序列化失败。**建议显式定义**。

---

## Q4：哪些字段不参与序列化？

**A：**
1. `static` 字段：属于类，不属于对象
2. `transient` 字段：显式标记不序列化
