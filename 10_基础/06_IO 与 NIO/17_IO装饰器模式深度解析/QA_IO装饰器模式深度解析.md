---
title: IO装饰器模式深度解析面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# IO装饰器模式深度解析

## Q1：Java IO 流体系中，哪些类是装饰器？哪些是被装饰者？

**A：**

| 类型 | 类名 | 装饰了什么 |
|------|------|-----------|
| **被装饰者（Component）** | InputStream、OutputStream、Reader、Writer | 基础组件 |
| **被装饰者（具体）** | FileInputStream、FileOutputStream、FileReader | 直接与数据源交互 |
| **装饰器基类（Decorator）** | FilterInputStream、FilterOutputStream、FilterReader、FilterWriter | 抽象装饰器 |
| **具体装饰器（Decorator）** | BufferedInputStream、BufferedOutputStream、BufferedReader、BufferedWriter | 添加缓冲功能 |
| **具体装饰器（Decorator）** | DataInputStream、DataOutputStream | 添加数据类型读写 |
| **具体装饰器（Decorator）** | InputStreamReader、OutputStreamWriter | 字节→字符转换 |

> **规律**：装饰器的类名通常是 `XxxInputStream` / `XxxOutputStream` / `XxxReader` / `XxxWriter`，而 FilterXxx 是它们的抽象父类。

---

## Q2：BufferedInputStream 为什么能提升读取性能？

**A：**

**原始 FileInputStream**：每次 `read()` 都触发一次系统调用 read()，从磁盘读取。

**BufferedInputStream**：内置 `byte[] buffer`，一次性从底层流读取**一批字节**填充缓冲区，之后多次 `read()` 直接从缓冲区取数据，减少系统调用次数。

```java
// 原始方式：1000次read = 1000次系统调用
FileInputStream fis = new FileInputStream("big.txt");
while (fis.read() != -1) { }  // 每次读1个字节，1000次系统调用

// Buffered方式：1000次read = 1次系统调用（填充缓冲区）
BufferedInputStream bis = new BufferedInputStream(
    new FileInputStream("big.txt"));
while (bis.read() != -1) { }  // 第一次触发fill()填充缓冲区，后续从内存取
```

性能提升比例：**约 10-100 倍**（取决于缓冲区大小和数据特点）。

---

## Q3：装饰器模式和继承（子类继承）相比，有什么优势？

**A：**

| 维度 | 继承（子类继承） | 装饰器模式 |
|------|:---:|:---:|
| 扩展时机 | 编译时（写代码时决定） | 运行时（动态组合） |
| 功能组合数量 | 受限于继承层次 | 任意组合 |
| 类数量 | 指数级爆炸（n个功能=2^n个子类） | 线性增长（n个功能=n个装饰器） |
| 复用性 | 子类复用父类代码 | 装饰器复用被装饰对象代码 |
| 扩展方式 | 新增子类 | 新增装饰器类 |

```java
// 继承方式的问题：
// 需要 File+Buffered+Data = 1个子类
// 需要 File+Buffered+LineNumber = 又1个子类
// 如果有5种功能两两组合 = C(5,2)=10个子类！爆炸！

// 装饰器方式：自由叠加
InputStream in = new LineNumberInputStream(        // 装饰器A
    new DataInputStream(                            // 装饰器B
        new BufferedInputStream(                    // 装饰器C
            new FileInputStream("file.txt")        // 被装饰者
        )
    )
);
```

---

## Q4：装饰器模式和策略模式有什么区别？

**A：**

| 维度 | 装饰器模式 | 策略模式 |
|------|:---:|:---:|
| 目的 | 动态添加功能（叠加） | 切换算法/行为 |
| 数量 | 可以叠加多个 | 同一时刻只用一个 |
| 关系 | 包装（has-a） | 替换（is-a） |
| 运行时 | 可运行时动态添加 | 通常编译时选择 |
| 示例 | BufferedInputStream 包裹 FileInputStream | Collections.sort(list, comparator) |

简单区分：**策略模式是"换算法"，装饰器模式是"加功能"**。

---

## Q5：FilterInputStream 的 in 字段为什么用 volatile 修饰？

**A：**

```java
protected volatile InputStream in;
```

volatile 的作用是**保证可见性**：当多个线程共享同一个 BufferedInputStream 实例时，一个线程修改了 `in` 引用的值，其他线程能立即看到最新值。

但更重要的原因是：**FilterInputStream 本身线程安全取决于被装饰的组件**。如果被装饰的是 BufferedInputStream，则整个链路是线程安全的（虽然性能会下降）。通常不推荐在多线程间共享同一个流对象。

---

## Q6：装饰器模式的调用链是怎样的？

**A：**

以 `DataInputStream + BufferedInputStream + FileInputStream` 为例：

```
readInt()
  └─ DataInputStream.readInt()
       └─ read() × 4
            └─ BufferedInputStream.read()
                 └─ (缓冲区未满) → fill()
                      └─ FileInputStream.read()  ← 底层系统调用
```

数据流方向：

```
FileInputStream（磁盘）→ BufferedInputStream（缓冲区）→ DataInputStream（类型转换）→ 程序
```

每层装饰器只做一件事：**Buffered**管缓冲，**Data**管类型转换，职责清晰。

---

## Q7：装饰器模式有什么缺点？如何避免？

**A：**

**缺点**：

1. **调试困难**：层层嵌套，单步调试需要逐层进入
2. **顺序敏感**：`new DataInputStream(new BufferedInputStream(...))` 和反过来效果不同
3. **配置复杂**：大量嵌套导致代码难以阅读
4. **类型问题**：装饰器不改变接口类型，外层装饰器只实现了底层组件的接口

**如何避免**：

1. **限制嵌套深度**：一般不超过 3-4 层
2. **使用工具类封装常见组合**：Java 9+ 的 `InputStream.of()` 可以简化

```java
// Java 9+ 简化方式
InputStream in = InputStream.nullInputStream();  // 空流
InputStream in = InputStream.readAllBytes();     // 读全部
```

3. **考虑 Builder 模式**：构建复杂 IO 链时使用 Builder 封装
