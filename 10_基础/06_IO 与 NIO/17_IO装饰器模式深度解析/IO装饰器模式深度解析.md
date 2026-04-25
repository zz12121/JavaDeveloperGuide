---
title: IO装饰器模式深度解析
tags:
  - Java/IO
  - 原理型
  - 设计模式
module: 06_IO与NIO
created: 2026-04-25
---

# IO装饰器模式深度解析

## 装饰器模式概述

### 什么是装饰器模式
- 装饰器模式（Decorator Pattern）：动态地给对象添加额外职责，比继承更灵活
- IO流体系大量使用装饰器模式

### 核心类结构

```
Component（抽象组件）
  └── InputStream / OutputStream / Reader / Writer

Decorator（抽象装饰器）
  └── FilterInputStream / FilterOutputStream / FilterReader / FilterWriter

ConcreteDecorator（具体装饰器）
  └── BufferedInputStream、DataInputStream、PushbackInputStream...
  └── BufferedOutputStream、BufferedWriter...
```

## Java IO 中的装饰器实现

### FilterInputStream（装饰器基类）

```java
public class FilterInputStream extends InputStream {
    protected volatile InputStream in;  // 被装饰的组件

    protected FilterInputStream(InputStream in) {
        this.in = in;
    }

    public int read() throws IOException {
        return in.read();  // 默认委托给被装饰的组件
    }
}
```

### 具体装饰器的三种类型

**1. 功能增强型（BufferedInputStream）**

```java
public class BufferedInputStream extends FilterInputStream {
    private byte[] buffer;  // 新增：缓冲区

    public int read() throws IOException {
        if (pos >= count) {
            fill();  // 一次性从底层流读取多个字节到缓冲区
            if (pos >= count) return -1;
        }
        return buffer[pos++];  // 从缓冲区读取
    }
}
```

**2. 数据转换型（DataInputStream）**

```java
public class DataInputStream extends FilterInputStream {
    public int readInt() throws IOException {
        return (read() << 24) | (read() << 16)
             | (read() << 8) | read();  // 字节→int转换
    }
}
```

**3. 特殊功能型（PushbackInputStream）**

```java
public class PushbackInputStream extends FilterInputStream {
    private byte[] buf;
    private int pos;

    public void unread(int b) throws IOException {
        if (pos >= buf.length)
            throw new IOException("Pushback buffer full");
        buf[pos++] = (byte) b;  // 把字节推回流中
    }
}
```

## 装饰器模式的IO链式调用

### 典型写法

```java
// 层层嵌套装饰：文件 → 字节流 → 缓冲流 → 数据流
InputStream in = new DataInputStream(  // 数据类型转换
    new BufferedInputStream(           // 缓冲提升性能
        new FileInputStream("data.txt")  // 文件输入
    )
);
int i = in.readInt();  // 直接读 int
```

### 组合方式总结

| 外层装饰器 | 内层流 | 新增功能 |
|-----------|--------|---------|
| BufferedInputStream | FileInputStream | 缓冲读取 |
| DataInputStream | BufferedInputStream | 读基本类型 |
| BufferedReader | FileReader | 按行读取+缓冲 |
| InputStreamReader | Socket.getInputStream | 字节→字符转换 |
| BufferedWriter | FileWriter | 缓冲写入 |

### 输出流链

```java
// 打印流：缓冲 + 自动刷新 + 格式化
PrintWriter out = new PrintWriter(
    new BufferedWriter(
        new FileWriter("output.txt")
    ),
    true  // autoFlush
);
```

## 装饰器 vs 继承

| 维度 | 继承 | 装饰器 |
|------|:---:|:---:|
| 灵活性 | 编译时固定 | 运行时动态组合 |
| 类数量 | O(n²) 爆炸 | O(n) 线性 |
| 功能叠加 | 难以叠加 | 随意叠加 |
| 复用性 | 高（子类复用父类） | 低（装饰器专用于组件） |
| 示例 | PrintStream | BufferedOutputStream |

### 继承方式的问题（类爆炸）

如果用继承实现所有组合：FileInputStream + Buffered + LineNumber + Data + ...
需要 2^n 个子类！

## 装饰器模式的优缺点

### 优点

1. **动态扩展**：运行时自由组合装饰器
2. **单一职责**：每个装饰器只关注一个功能
3. **开闭原则**：对扩展开放，对修改关闭
4. **可替换性**：可以移除任意装饰器，不影响其他部分

### 缺点

1. **调试困难**：层层嵌套，调用链长
2. **顺序敏感**：装饰器顺序影响行为
3. **配置复杂**：容易写出过于复杂的嵌套

## 关联知识点

