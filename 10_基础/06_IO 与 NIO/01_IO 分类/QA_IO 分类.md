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

Java IO 流从**两大维度**进行分类，组合后形成四大基类体系：

### 维度一：按数据单位

| 分类 | 基类 | 单位 | 说明 |
|------|------|------|------|
| **字节流** | `InputStream` / `OutputStream` | 8位（1字节） | 最基础的流，以字节为单位读写，不涉及编码转换 |
| **字符流** | `Reader` / `Writer` | 16位（1个char） | 以字符为单位读写，内部自动完成字节→字符的编码解码 |

### 维度二：按流向

| 分类 | 基类 | 方向 |
|------|------|------|
| **输入流** | `InputStream` / `Reader` | 从外部数据源 → 程序内存（读） |
| **输出流** | `OutputStream` / `Writer` | 从程序内存 → 外部目标（写） |

### 四大基类

```
                ┌─────────────┐
                │   输入/输出    │
                └──────┬──────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            │
    ┌──────────┐  ┌──────────┐     │
    │ 字节流     │  │ 字符流     │     │
    │InputStream│  │  Reader   │     │
    │OutputStream│  │  Writer   │     │
    └──────────┘  └──────────┘     │
                                  │
                    字节流是根基，字符流封装字节流
```

### 设计思想

Java IO 采用**装饰器模式（Decorator Pattern）**：
- **节点流（Node Stream）**：直接与数据源连接的流，如 `FileInputStream`、`ByteArrayInputStream`
- **处理流（Processing Stream）**：包装节点流，添加额外功能，如 `BufferedInputStream`、`DataInputStream`

这种分层设计使得功能可以灵活叠加：`new DataInputStream(new BufferedInputStream(new FileInputStream("file")))`

---

## Q2：字节流和字符流的区别？

**A：**

### 核心区别对比表

| **维度** | **字节流** | **字符流** |
|----------|-----------|-----------|
| **基类** | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| **操作单位** | 1 字节（8 bit） | 1 字符（16 bit，char） |
| **中文处理** | 需要手动指定编码，否则乱码 | 自动处理编码转换 |
| **缓冲机制** | 无内置缓冲（需手动加 Buffered） | 部分实现有内置缓冲（如 BufferedReader 的 readLine()） |
| **适用场景** | 二进制文件、网络传输、所有非文本数据 | 文本文件、XML/JSON/CSV 等 |
| **性能** | 直接操作底层字节，无转换开销 | 多一层编解码，略有开销 |
| **典型类** | FileInputStream, ByteArrayOutputStream | FileReader, PrintWriter |

### 为什么需要字符流？

字符流的出现是为了解决**文本数据的编码问题**。在 Java 内部，字符串使用 UTF-16 编码（char = 2字节），但文件系统中的文本可能使用各种编码（GBK、UTF-8、ISO-8859-1等）。字符流通过 `InputStreamReader` / `OutputStreamWriter` 作为桥梁：

```java
// 字节流读取中文 —— 容易乱码
FileInputStream fis = new FileInputStream("chinese.txt");
byte[] buffer = new byte[1024];
int len = fis.read(buffer);
// 如果文件是 UTF-8 编码，直接 new String(buffer) 可能截断中文字符

// 字符流读取中文 —— 自动处理编码
Reader reader = new InputStreamReader(new FileInputStream("chinese.txt"), StandardCharsets.UTF_8);
char[] charBuffer = new char[512];
int charLen = reader.read(charBuffer);
// 正确地将字节转换为字符
```

### 字符流的本质

**字符流底层仍然基于字节流**。`Reader` 和 `Writer` 是高层抽象，实际工作流程为：

```
文件(字节) → InputStream → InputStreamReader(解码) → Reader(char) → 程序
程序 → Writer(char) → OutputStreamWriter(编码) → OutputStream → 文件(字节)
```

`InputStreamReader` 内部维护了一个 `StreamDecoder`，负责将字节流按照指定字符集解码为字符；`OutputStreamWriter` 内部使用 `StreamEncoder` 将字符编码为字节。

### 选择原则

> **不确定时用字节流，确定是文本时用字符流。**

- 图片、视频、音频、压缩包、Class 文件 → **必须用字节流**
- txt、csv、json、xml、properties、log 文件 → **推荐用字符流**
- 网络 Socket 通信 → 通常用**字节流**（协议层面处理）

---

## Q3：什么时候用字节流，什么时候用字符流？

**A：**

### 一句话判断法

> **看数据类型：二进制用字节流，纯文本用字符流。**

### 详细选择指南

#### 必须使用字节流的场景

```java
// 1. 复制图片/音频/视频文件
try (InputStream in = new FileInputStream("photo.jpg");
     OutputStream out = new FileOutputStream("copy.jpg")) {
    byte[] buf = new byte[8192];
    int len;
    while ((len = in.read(buf)) != -1) {
        out.write(buf, 0, len);
    }
}

// 2. 读写 Class 文件或任意二进制数据
byte[] classBytes = Files.readAllBytes(Paths.get("MyClass.class"));

// 3. 网络通信（Socket 传输）
Socket socket = new Socket("localhost", 8080);
InputStream netIn = socket.getInputStream();
OutputStream netOut = socket.getOutputStream();

// 4. 对象序列化（ObjectOutputStream 底层是字节流）
ObjectOutputStream oos = new ObjectOutputStream(
    new FileOutputStream("object.dat")
);
oos.writeObject(user);

// 5. 加密/解密操作（操作的是原始字节）
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
byte[] encrypted = cipher.doFinal(plainBytes);
```

#### 推荐使用字符流的场景

```java
// 1. 读取文本配置文件（UTF-8）
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("config.properties"), "UTF-8"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// 2. 写入 JSON/XML 数据
try (Writer writer = new OutputStreamWriter(
        new FileOutputStream("data.json"), StandardCharsets.UTF_8)) {
    writer.write("{\"name\":\"张三\"}");
}

// 3. 日志写入
PrintWriter logWriter = new PrintWriter(
    new FileWriter("app.log", true)  // append模式
);
logWriter.println("[ERROR] 2026-04-25 发生异常");

// 4. CSV 文件读写
try (BufferedReader csvReader = new BufferedReader(new FileReader("data.csv"))) {
    // 按行读取，自动处理换行符
}
```

#### 特殊情况讨论

**混合场景**——HTTP 协议：请求头是文本（可用字符流），请求体可能是二进制（如文件上传，需切换到字节流）。通常 HTTP 框架（如 Servlet）会提供 `getInputStream()`（字节流）和 `getReader()`（字符流），但**同一个请求中两者不能混用**，调用其中一个后另一个再调用会抛 `IllegalStateException`。

**NIO 中的 ByteBuffer vs CharBuffer**：NIO 统一了字节和字符的操作，`CharsetDecoder` 和 `CharsetEncoder` 负责转换，不再像传统 IO 那样分为两套独立的类层次结构。

---

## Q4：常见的字符编码有哪些？Java 中如何正确处理字符编码问题？

**A：**

### 主流字符编码一览

| **编码名称** | **全称** | **字符集范围** | **变长规则** | **典型用途** |
|-------------|---------|--------------|------------|------------|
| **ASCII** | American Standard Code for Information Interchange | 英文字母+数字+符号（128个） | 固定1字节 | 古老协议、英文环境 |
| **ISO-8859-1** (Latin-1) | 西欧语言扩展 ASCII | 256个字符 | 固定1字节 | HTTP 默认头编码、西欧语言 |
| **GBK** | 国标扩展 | 中文 + 全部ASCII | 1~2字节（中文2字节） | 中国大陆传统系统 |
| **GB2312** | 国标简化版 | 简体中文6763字 + ASCII | 1~2字节 | 已淘汰，仅旧系统 |
| **GB18030** | 国标最新版 | 全部汉字+少数民族文字 | 1/2/4字节 | 政府要求合规系统 |
| **UTF-8** | 8-bit Unicode Transformation Format | Unicode全部字符 | 1~4字节（英文1字节，中文3字节） | **互联网标准首选** |
| **UTF-16** | 16-bit Unicode TF | Unicode全部字符 | 大多2字节（部分 surrogate 4字节） | Java 内部 char 编码 |
| **UTF-32** | 32-bit Unicode TF | Unicode全部字符 | 固定4字节 | 极少使用 |

### Java 中的编码处理

Java 内部使用 **UTF-16** 存储 `String`（每个 char 16位）。当与外界交互时（文件、网络、数据库），需要进行编码转换：

```java
public class EncodingDemo {

    public static void main(String[] args) throws Exception {
        String chinese = "你好世界Hello";

        // ===== 1. 获取不同编码的字节数组 =====
        byte[] utf8Bytes = chinese.getBytes(StandardCharsets.UTF_8);
        byte[] gbkBytes = chinese.getBytes("GBK");
        // UTF-8: 中文每个字3字节 + 英文5字节 = 17字节
        // GBK:   中文每个字2字节 + 英文5字节 = 13字节
        System.out.println("UTF-8长度: " + utf8Bytes.length); // 17
        System.out.println("GBK长度:   " + gbkBytes.length);   // 13

        // ===== 2. 错误示范：编码不一致导致乱码 =====
        // 用 GBK 编码的字节，用 UTF-8 解码 → 乱码
        String garbled = new String(gbkBytes, StandardCharsets.UTF_8);
        System.out.println("乱码结果: " + garbled); // ���ã����...

        // ===== 3. 正确做法：编码和解码保持一致 =====
        String correct = new String(gbkBytes, "GBK");
        System.out.println("正确还原: " + correct); // 你好世界Hello

        // ===== 4. Java 源文件编码与运行时编码 =====
        // javac -encoding UTF-8 Xxx.java  指定源文件编码
        // 或者在 IDEA 中设置 File Encodings → Global/Project/Properties 都设为 UTF-8

        // ===== 5. 推荐方式：始终显式指定 Charset =====
        try (Reader reader = new InputStreamReader(
                new FileInputStream("test.txt"), StandardCharsets.UTF_8)) {
            // ...
        }

        // ===== 6. BOM（Byte Order Mark）问题 =====
        // Windows 记事本保存 UTF-8 时可能带 BOM（EF BB BF）
        // Java 的 StandardCharsets.UTF_8 不会主动写入/跳过 BOM
        // 如需处理带 BOM 的文件，可手动检测并跳过前3字节
    }
}
```

### 常见编码陷阱

#### 陷阱1：默认编码导致的跨平台问题

```java
// ❌ 危险！依赖平台默认编码
String s = new String(bytes);           // 等价于 new String(bytes, Charset.defaultCharset())
byte[] b = str.getBytes();              // 同上，Windows通常是GBK，Linux是UTF-8

// ✅ 安全！始终显式指定
String s = new String(bytes, StandardCharsets.UTF_8);
byte[] b = str.getBytes(StandardCharsets.UTF_8);
```

#### 陷阱2：String.length() ≠ 实际字符数

```java
String emoji = "🔥";
System.out.println(emoji.length());      // 输出 2（Java char 是 UTF-16，emoji 需要 surrogate pair）
System.out.println(emoji.codePointCount(0, emoji.length())); // 输出 1（正确的"字符"数）
// UTF-8 中 🔥 占 4 个字节
System.out.println(emoji.getBytes(StandardCharsets.UTF_8).length); // 4
```

#### 陷阱3：URL 编码与字符编码混淆

```java
// URL 编码 ≠ 字符编码！URL编码是将特殊字符转为 %XX 格式
// 但 URL 编码前需要先将字符串转为某编码的字节
String encoded = URLEncoder.encode("你好", "UTF-8"); // → %E4%BD%A0%E5%A5%BD
```

#### 陷阱4：数据库连接字符串未指定编码

```java
# JDBC 连接 URL 必须指定 characterEncoding，否则可能用服务器默认编码
jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
```

### 最佳实践总结

1. **统一使用 UTF-8**：项目内所有文件、数据库、网络接口统一 UTF-8
2. **永远不要依赖默认编码**：每次都显式指定 `StandardCharsets.UTF_8`
3. **IDE 设置一致**：IDEA/VSCode/Eclipse 的文件编码全部设置为 UTF-8
4. **注意 JVM 参数**：启动时可设置 `-Dfile.encoding=UTF-8`
5. **HTTP 请求/响应**：`Content-Type: text/plain; charset=utf-8`
6. **BOM 问题**：避免 Windows 记事本编辑后引入 BOM，或使用工具去除

---

## Q5：Java IO 流体系的完整继承结构是怎样的？

**A：**

### InputStream 体系（字节输入流）

```
InputStream (抽象类)
├── ByteArrayInputStream       // 字节数组输入
├── FileInputStream            // 文件输入
├── FilterInputStream (抽象装饰器)
│   ├── BufferedInputStream    // 缓冲输入 ★★★
│   ├── DataInputStream        // 基本类型数据输入
│   └── PushbackInputStream    // 回退流（可unread）
├── ObjectInputStream          // 对象反序列化
├── PipedInputStream           // 管道输入（线程间通信）
├── SequenceInputStream        // 序列合并多个流
├── StringBufferInputStream    // @Deprecated（已被 StringReader 替代）
└── AudioInputStream           // 音频输入（javax.sound.sampled）
```

### OutputStream 体系（字节输出流）

```
OutputStream (抽象类)
├── ByteArrayOutputStream      // 字节数组输出
├── FileOutputStream           // 文件输出
├── FilterOutputStream (抽象装饰器)
│   ├── BufferedOutputStream   // 缓冲输出 ★★★
│   ├── DataOutputStream       // 基本类型数据输出
│   └── PrintStream            // 打印流（System.out 就是它）
├── ObjectOutputStream         // 对象序列化
├── PipedOutputStream          // 管道输出
└── ...
```

### Reader 体系（字符输入流）

```
Reader (抽象类)
├── BufferedReader             // 缓冲字符输入 ★★★（支持readLine()）
├── InputStreamReader          // 字节→字符桥接器（关键类！）
│   └── FileReader             // 文件字符输入（简化版InputStreamReader）
├── StringReader               // 字符串输入
├── CharArrayReader            // 字符数组输入
├── PipedReader                // 管道字符输入
├── FilterReader (抽象)
│   └── PushbackReader         // 回退字符流
└── LineNumberReader           // 行号追踪（@Deprecated，用BufferedReader替代）
```

### Writer 体系（字符输出流）

```
Writer (抽象类)
├── BufferedWriter             // 缓冲字符输出 ★★★
├── OutputStreamWriter         // 字符→字节桥接器（关键类！）
│   └── FileWriter             // 文件字符输出（简化版OutputStreamWriter）
├── StringWriter               // 字符串输出
├── CharArrayWriter            // 字符数组输出
├── PrintWriter                // 打印字符流（支持格式化输出）★★★
├── PipedWriter                // 管道字符输出
└── FilterWriter (抽象)
```

### 关键设计模式

整个 IO 体系的核心设计模式是 **装饰器模式（Decorator Pattern）**：

```java
// 装饰器的嵌套组合示例：
// 功能：从文件读取基本类型数据，带缓冲
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.bin")
    )
);

// 解析：
// 1. FileInputStream     — 节点流：打开文件
// 2. BufferedInputStream — 装饰器：添加8KB缓冲区
// 3. DataInputStream     — 装饰器：添加 readInt/readLong 等方法
```

装饰器模式的优点：**可以动态、灵活地组合功能**，而不需要创建大量子类（如果用继承，`BufferedFileDataInput`、`BufferedFileDataOutput`... 组合爆炸）。

---

## Q6：什么是标准的 IO 关闭模式？try-with-resources 的原理是什么？

**A：**

### 传统关闭方式的问题

```java
// ❌ 传统方式：繁琐且容易出错
InputStream in = null;
try {
    in = new FileInputStream("test.txt");
    // ... 使用in
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (in != null) {
        try {
            in.close();  // close本身也可能抛异常
        } catch (IOException e) {
            e.printStackTrace();  // 吞掉了原始异常信息
        }
    }
}
```

**问题**：
1. finally 中 `close()` 本身也可能抛异常，导致**原始异常被覆盖**
2. 多个资源时代码嵌套极深
3. 容易忘记关闭，造成资源泄漏

### try-with-resources（JDK 7+）

```java
// ✅ JDK 7+ 推荐：自动关闭，简洁安全
static String readFile(String path) throws IOException {
    try (InputStream in = new FileInputStream(path)) {
        return new String(in.readAllBytes(), StandardCharsets.UTF_8);
    }
}
```

### 工作原理

`try-with-resources` 语法糖编译后的等价代码：

```java
// 编译器生成的等价代码（简化版）：
static String readFile(String path) throws IOException {
    InputStream in = new FileInputStream(path);
    try {
        return new String(in.readAllBytes(), StandardCharsets.UTF_8);
    } catch (Throwable var3) {  // 捕获 try 块中的异常
        if (in != null) {
            try {
                in.close();  // 尝试关闭资源
            } catch (Throwable var4) {  // close 也抛了异常
                var3.addSuppressed(var4);  // ⭐ 关键：close异常被"压制"，附加到主异常上
            }
        }
        throw var3;  // 抛出原始异常（包含被压制的close异常）
    } finally {  // 正常执行完也会走到这里关闭
        if (in != null) {
            in.close();
        }
    }
}
```

核心要点：
1. 实现 `AutoCloseable`（或 `Closeable`）接口的资源才能用
2. **关闭顺序与声明顺序相反**（先声明的后关闭）
3. **Suppressed Exception**：try块异常 + close异常时，close异常不会被丢弃，而是通过 `getSuppressed()` 获取
4. 可以同时管理多个资源：

```java
// 多资源管理
try (InputStream in = new FileInputStream("src.txt");
     OutputStream out = new FileOutputStream("dest.txt")) {
    out.write(in.readAllBytes());
}  // 先关out，后关in（声明逆序）
```
