---
title: 节点流 vs 处理流面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# 节点流 vs 处理流

## Q1：节点流和处理流有什么区别？

**A：**
- **节点流**：直接连接数据源（文件、内存、网络），可以单独使用，如 `FileInputStream`
- **处理流**：不能独立使用，必须包装节点流，提供额外功能（缓冲、转换、序列化），如 `BufferedInputStream`
处理流的设计基于**装饰器模式**，可以多层嵌套组合。

---

## Q2：为什么 IO 流使用装饰器模式而不是继承？

**A：**
如果用继承，功能组合会导致类爆炸。例如 3 种节点流 × 3 种增强功能 = 9 个子类。装饰器模式可以运行时任意组合，只需要 3 个装饰类 + 原有类即可。

---

## Q3：请举一个多层嵌套的例子？

**A：**
```java
BufferedReader br = new BufferedReader(
    new InputStreamReader(
        new FileInputStream("data.txt"), "UTF-8"
    )
);
```
`FileInputStream`（节点流）→ `InputStreamReader`（字节转字符）→ `BufferedReader`（缓冲+按行读取）
