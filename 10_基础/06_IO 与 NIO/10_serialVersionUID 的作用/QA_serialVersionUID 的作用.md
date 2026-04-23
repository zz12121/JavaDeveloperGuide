---
title: serialVersionUID 面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# serialVersionUID

## Q1：serialVersionUID 有什么作用？

**A：**
用于序列化版本校验。反序列化时 JVM 比较字节流中的 UID 和当前类的 UID，不一致则抛 `InvalidClassException`。

## Q2：如果不定义 serialVersionUID 会怎样？

**A：**
JVM 会根据类结构自动生成。类一旦修改（增删字段等），UID 会变化，导致之前序列化的数据无法反序列化。**所以建议始终显式定义**。

## Q3：显式定义 serialVersionUID 后，类新增字段还能反序列化吗？

**A：**
可以。新增字段反序列化时会被赋默认值。删除字段时多余数据会被忽略。只要 UID 不变就是兼容的。

## Q4：什么修改会导致不兼容？

**A：**
字段改名、修改字段类型、修改类名等结构性变化。新增/删除字段是兼容的。
