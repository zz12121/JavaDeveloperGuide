---
title: transient 关键字面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# transient 关键字

## Q1：transient 关键字的作用？

**A：**
`transient` 修饰的字段不参与序列化。反序列化时，该字段被恢复为类型的默认值（int → 0，引用类型 → null）。

## Q2：transient 修饰的字段反序列化后是什么值？

**A：**
类型的默认值。`int` 为 0，`boolean` 为 false，对象引用为 null。

## Q3：什么场景下会用到 transient？

**A：**
1. **敏感信息**：密码、密钥不应被序列化到文件
2. **不可序列化的类型**：`Thread`、`Socket`、`Connection` 等
3. **临时数据**：缓存、运行时计算结果

## Q4：transient 和 static 都不参与序列化，有什么区别？

**A：**
- `transient` 字段反序列化后为**默认值**
- `static` 字段反序列化后保持**类当前的静态值**（序列化/反序列化过程不涉及 static 字段）
