---
title: IO 分类
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# IO 分类

## 两大维度

### 按数据单位分

| 类型 | 基类 | 说明 | 适用场景 |
|------|------|------|---------|
| **字节流** | `InputStream` / `OutputStream` | 以字节为单位（8 bit） | 二进制文件（图片、视频、音频、压缩包） |
| **字符流** | `Reader` / `Writer` | 以字符为单位（16 bit） | 文本文件（txt、csv、json、xml） |

### 按流向分

| 类型 | 说明 |
|------|------|
| **输入流** | 从数据源 → 程序（读） |
| **输出流** | 从程序 → 数据源（写） |

## 四大基类

```
          输入流                    输出流
字节流   InputStream            OutputStream
字符流   Reader                  Writer
```

## 完整分类体系

```
InputStream（字节输入）
  ├── FileInputStream        节点流：读取文件
  ├── ByteArrayInputStream  节点流：读取字节数组
  ├── BufferedInputStream   处理流：缓冲
  ├── DataInputStream       处理流：基本数据类型
  └── ObjectInputStream     处理流：反序列化

OutputStream（字节输出）
  ├── FileOutputStream       节点流：写入文件
  ├── ByteArrayOutputStream 节点流：写入字节数组
  ├── BufferedOutputStream  处理流：缓冲
  ├── DataOutputStream      处理流：基本数据类型
  └── ObjectOutputStream    处理流：序列化

Reader（字符输入）
  ├── FileReader            节点流：读取文件
  ├── StringReader          节点流：读取字符串
  ├── BufferedReader        处理流：缓冲（按行读取）
  └── InputStreamReader     处理流：字节→字符（桥梁）

Writer（字符输出）
  ├── FileWriter            节点流：写入文件
  ├── StringWriter          节点流：写入字符串
  ├── BufferedWriter        处理流：缓冲
  └── OutputStreamWriter    处理流：字符→字节（桥梁）
```

## 节点流 vs 处理流

| 类型 | 说明 | 能否单独使用 |
|------|------|------------|
| **节点流** | 直接与数据源交互（文件、数组、管道） | ✅ 可以 |
| **处理流** | 包装节点流，提供额外功能（缓冲、转换、序列化） | ❌ 必须包装节点流 |

## 字节流 vs 字符流的区别

| 维度 | 字节流 | 字符流 |
|------|--------|--------|
| 单位 | 1 字节（byte） | 1 字符（char，2 字节） |
| 中文处理 | 可能乱码（需手动处理编码） | 自动处理编码 |
| 缓冲 | 无内置缓冲 | 内置缓冲（BufferedReader/Writer） |
| 底层 | 直接操作字节 | 底层仍基于字节流 + 编码转换 |
| 性能 | 二进制场景更快 | 文本场景更方便 |

## 关联知识点
