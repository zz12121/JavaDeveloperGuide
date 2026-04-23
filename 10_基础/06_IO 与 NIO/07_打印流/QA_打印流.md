---
title: 打印流
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# 打印流

## Q1：System.out 是什么类型？

**A：**
`System.out` 是 `PrintStream` 类型，是字节打印流。JVM 启动时自动创建，关联到控制台输出。

## Q2：PrintStream 和 PrintWriter 的区别？

**A：**
- `PrintStream` 是字节流（继承 `OutputStream`），`System.out` 就是它
- `PrintWriter` 是字符流（继承 `Writer`），推荐用于文本输出
- 两者都**不抛 IOException**，通过 `checkError()` 检查错误
