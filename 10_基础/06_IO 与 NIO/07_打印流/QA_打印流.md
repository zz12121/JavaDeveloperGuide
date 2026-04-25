---
title: 打印流面试题
tags:
  - Java/IO
  - 原理型
  - 问答
module: 06_IO与NIO
created: 2026-04-25
---

# 打印流

## Q1：PrintStream 和 PrintWriter 的区别？

**A：**

| 维度 | PrintStream | PrintWriter |
|------|-------------|-------------|
| 底层类型 | OutputStream（字节流） | Writer（字符流） |
| 构造参数 | OutputStream/File/String | Writer/File/String/OutputStream |
| 自动刷新 | 可设置 autoflush | 可设置 autoflush |
| 编码处理 | 字节方式 | 支持字符编码 |
| 多线程安全 | 否（已同步方法） | 否（已同步方法） |
| JDK 1.4+ | PrintWriter 更推荐 | 推荐使用 |

```java
// PrintStream（字节流）
PrintStream out = new PrintStream(new FileOutputStream("out.txt"));

// PrintWriter（字符流，更推荐）
PrintWriter pw = new PrintWriter(new OutputStreamWriter(new FileOutputStream("out.txt"), "UTF-8"));
```

**推荐使用 PrintWriter**，因为它支持字符编码指定，避免乱码。

---

## Q2：PrintStream 的自动刷新机制？

**A：**
PrintStream 在以下情况自动刷新缓冲区：
1. 写入换行符 `\n` 时（需要设置 `autoflush=true`）
2. 调用 `println()` 方法时（每次写一行）
3. 调用 `printf()` 时（格式化输出）

```java
// 自动刷新 PrintStream
PrintStream ps = new PrintStream(new FileOutputStream("log.txt"), true);  // true=autoflush

ps.println("Line 1");   // 刷新
ps.print("No flush");  // 不刷新
ps.printf("%s%n", "Line 2");  // 格式化换行，刷新
```

---

## Q3：PrintStream 和 System.out 的关系？

**A：**
`System.out` 就是 PrintStream 的实例：

```java
// System.out 源码
public final class System {
    public static final PrintStream out = null;  // 由 JVM 初始化
}

// System.setOut() 重定向标准输出
PrintStream fileOut = new PrintStream("output.txt");
System.setOut(fileOut);
System.out.println("写入文件而非控制台");
```

---

## Q4：PrintStream 会抛出 IOException 吗？

**A：**
**不会**。PrintStream 的所有写方法都不会抛出 IOException，而是将错误状态记录在内部：

```java
public class PrintStream extends FilterOutputStream {
    private boolean trouble = false;

    @Override
    public void write(int b) {
        try {
            out.write(b);
            if (out.flushError()) {
                trouble = true;
            }
        } catch (IOException x) {
            trouble = true;
        }
    }

    // 永远不抛 IOException，但可以检查错误状态
    public boolean checkError() {
        return trouble;
    }
}

// 正确做法
PrintStream ps = new PrintStream("file.txt");
ps.println("test");
ps.checkError();  // 检查是否有错误
```
