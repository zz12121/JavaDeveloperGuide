---
title: 节点流 vs 处理流
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# 节点流 vs 处理流（装饰器模式）

## 核心区别

| 维度 | 节点流 | 处理流 |
|------|--------|--------|
| 定义 | 直接与数据源（文件、内存、网络）交互 | 包装节点流，提供额外功能 |
| 能否单独使用 | ✅ 可以 | ❌ 必须包装节点流 |
| 典型代表 | `FileInputStream`、`FileReader` | `BufferedInputStream`、`BufferedReader` |

## 装饰器模式

IO 流的设计基于**装饰器模式（Decorator Pattern）**：

```
节点流（核心）        处理流（装饰）          处理流（装饰）
FileInputStream  →  BufferedInputStream  →  DataInputStream
    ↓                    ↓                     ↓
  读取文件            添加缓冲区          读取基本数据类型
```

### 特点

1. **不改变原有功能**：`BufferedInputStream` 仍然是一个 `InputStream`，只是增强了性能
2. **可以多层嵌套**：可以叠加多个处理流
3. **灵活组合**：按需组合功能

## 代码示例

### 单层装饰

```java
// 节点流直接使用
FileInputStream fis = new FileInputStream("data.bin");

// 处理流包装节点流（添加缓冲）
BufferedInputStream bis = new BufferedInputStream(fis);
```

### 多层装饰

```java
// 三层嵌套：文件 → 缓冲 → 数据类型
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.bin")
    )
);
int value = dis.readInt();  // 可以直接读取基本类型
```

### 字符流装饰

```java
// 字节流 → 字符转换 → 缓冲
BufferedReader br = new BufferedReader(
    new InputStreamReader(
        new FileInputStream("data.txt")
    )
);
String line = br.readLine();  // 按行读取
```

## 装饰器模式 vs 继承

| 维度 | 继承 | 装饰器 |
|------|------|--------|
| 扩展方式 | 子类化 | 包装组合 |
| 类数量 | 组合爆炸（缓冲+数据+文件 = 3×3 = 9 个类） | 线性增长（3 个类即可任意组合） |
| 灵活性 | 编译时确定 | 运行时动态组合 |

## IO 流中的装饰器关系

```
FilterInputStream（装饰器基类）
  ├── BufferedInputStream    添加缓冲
  ├── DataInputStream       读取基本数据类型
  ├── PushbackInputStream   支持回推
  └── ObjectInputStream     对象反序列化

FilterOutputStream（装饰器基类）
  ├── BufferedOutputStream  添加缓冲
  ├── DataOutputStream      写入基本数据类型
  ├── PrintStream           打印输出
  └── ObjectOutputStream   对象序列化
```

## 关联知识点