---
title: JDK 15新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Text Blocks 文本块

## 核心结论

Text Blocks（文本块）是 JDK 15 正式引入的特性（JDK 13 预览），使用三重双引号 `"""` 定义多行字符串，无需手动拼接和转义。编译器会**自动处理缩进**（将文本块内容与左侧 `"""` 对齐）和**尾随空格**。

---

## 深度解析

### 1. 基本语法

```java
// 传统方式：需要转义和换行拼接
String json = "{\n" +
    "  \"name\": \"Alice\",\n" +
    "  \"age\": 30\n" +
    "}";

// Text Blocks 方式
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
```

### 2. 缩进处理规则

编译器通过以下规则确定文本块的缩进：
1. 找到所有行的**公共前导空白**（最小缩进量）
2. 找到**尾随 `"""`** 的缩进位置
3. 取两者中的**较大值**作为要移除的缩进量

```java
// ✅ 推荐：尾随 """ 单独一行，缩进与内容对齐
String html = """
              <html>
                  <body>
                      <p>Hello</p>
                  </body>
              </html>
              """;
// 结果：每行都去除了公共缩进

// ❌ 尾随 """ 和内容不在同一缩进级别会导致意外缩进
```

### 3. 转义字符

```java
// 不需要转义双引号
String text = """
        He said "hello"
        """;

// 转义三重双引号（防止提前终止文本块）
String code = """
        String s = \\"""hello\\""";  // 转义三引号
        """;

// 续行符（防止在字符串中插入换行符）
String sql = """
        SELECT id, name \
        FROM users \
        WHERE status = 'active'
        """;
// 等价于："SELECT id, name FROM users WHERE status = 'active'"
```

### 4. 尾随空格

```java
// Text Blocks 会保留行尾有意添加的空格
String text = """
        line1   
        line2
        """.replace("   ", "   ");
```

> 编译器默认去除尾随空格。如需保留尾随空格，在行尾使用 `\0` 或 `\s`（JDK 14+）。

### 5. formatted() 结合

```java
// Text Blocks + formatted() = 模板引擎的平替
String template = """
        Dear %s,
        
        Thank you for your order #%d.
        
        Best regards,
        %s
        """.formatted(name, orderId, company);
```

### 6. 注意事项

| 注意点 | 说明 |
|--------|------|
| 空文本块 | `""""""`（两个连续的三引号）是空字符串，但中间至少要一个换行 |
| 不能单行 | `""" hello """` 编译错误，`"""` 后必须跟换行 |
| 行尾不能有不可见字符 | 可能影响缩进计算 |

---

## 代码示例

```java
// SQL 语句
String sql = """
        SELECT u.id, u.name, o.total
        FROM users u
        JOIN orders o ON u.id = o.user_id
        WHERE u.status = 'ACTIVE'
          AND o.created_at > ?
        ORDER BY o.total DESC
        LIMIT 10
        """;

// HTML 片段
String html = """
        <div class="card">
            <h2>%s</h2>
            <p>%s</p>
        </div>
        """.formatted(title, content);

// JSON（配合 Jackson/Gson 等）
String json = """
        {
            "name": "%s",
            "age": %d,
            "active": %b
        }
        """.stripIndent().formatted(name, age, active);
```

---

## 关联知识点
