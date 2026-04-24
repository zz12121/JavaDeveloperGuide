---
title: JDK 15新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Text Blocks 文本块

## Q1：Text Blocks 是什么？解决了什么问题？

**A**：Text Blocks 使用 `"""` 三重双引号定义多行字符串，是 JDK 15 正式引入的特性。解决了传统多行字符串需要大量拼接（`+`）和转义（`\n`、`\"`）的问题，大幅提升代码可读性。

---

## Q2：Text Blocks 中的缩进是怎么处理的？

**A**：编译器会自动移除公共前导缩进。具体规则：
1. 取所有非空行的最小缩进量（公共前导空白）
2. 取尾随 `"""` 的缩进位置
3. 取两者中较大值作为要移除的缩进

```java
String s = """
        hello
        """;
// 结果："hello\n"（公共缩进被移除）
```

---

## Q3：Text Blocks 中需要转义双引号吗？

**A**：单双引号和双双引号不需要转义，只有三重双引号才需要转义：

```java
"""
He said "hello"       // ✅ 不需要转义
""";

"""
String s = \\"""hi\\""";  // ✅ 三引号需要转义
""";
```

---

## Q4：如何在 Text Blocks 中避免换行？

**A**：在行尾使用 `\`（续行符）：

```java
String s = """
        SELECT id \
        FROM users \
        WHERE active = true
        """;
// 结果："SELECT id FROM users WHERE active = true\n"
```

---

## Q5：Text Blocks 从哪个 JDK 版本开始可用？

**A**：JDK 13 预览，JDK 14 二次预览，**JDK 15 正式发布**。生产环境至少需要 JDK 15。
