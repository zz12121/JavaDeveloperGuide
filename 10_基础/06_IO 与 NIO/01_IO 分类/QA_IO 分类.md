---
title: IO 分类面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-18
---

# IO 分类

## Q1：Java IO 流有几种分类方式？

**A：**
两大维度：
1. **按数据单位**：字节流（`InputStream`/`OutputStream`）和字符流（`Reader`/`Writer`）
2. **按流向**：输入流（读）和输出流（写）

组合后有四大基类：`InputStream`、`OutputStream`、`Reader`、`Writer`

---

## Q2：字节流和字符流的区别？

**A：**

| 维度 | 字节流 | 字符流 |
|------|--------|--------|
| 单位 | 1 字节 | 2 字节（char） |
| 中文 | 可能乱码 | 自动处理 |
| 缓冲 | 无 | 有 |
| 场景 | 二进制文件 | 文本文件 |

字符流底层仍然基于字节流，通过 `InputStreamReader`/`OutputStreamWriter` 做编码转换。

---

## Q3：什么时候用字节流，什么时候用字符流？

**A：**
- **字节流**：处理二进制数据（图片、视频、音频、压缩包、网络传输）
- **字符流**：处理文本数据（txt、csv、json、xml）
- 原则：不确定时用字节流，字节流是最通用的
