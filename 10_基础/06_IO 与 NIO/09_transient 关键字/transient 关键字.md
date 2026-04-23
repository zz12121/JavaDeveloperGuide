---
title: transient 关键字
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# transient 关键字

## 作用

`transient` 关键字标记的字段**不参与序列化**，反序列化时该字段为默认值。

## 使用示例

```java
public class User implements Serializable {
    private String name;
    private transient int age;           // 不序列化
    private transient String password;   // 敏感信息不序列化

    // 序列化后：
    // name = "Tom"     ✅ 正常序列化
    // age = 0           ❌ 不序列化，反序列化后为默认值 0
    // password = null    ❌ 不序列化
}
```

## 反序列化后的值

被 `transient` 修饰的字段在反序列化后会是**类型的默认值**：

| 类型 | 默认值 |
|------|--------|
| `int` / `long` / `double` 等 | `0` / `0L` / `0.0` |
| `boolean` | `false` |
| 引用类型（`String`、对象等） | `null` |

## 常见使用场景

1. **敏感信息**：密码、密钥不应序列化到文件
2. **不需要持久化的字段**：缓存、临时计算结果
3. **不可序列化的字段**：`Thread`、`Socket`、`Connection` 等无法序列化的类型

```java
public class DatabaseConfig implements Serializable {
    private String url;
    private String username;
    private transient String password;  // 不序列化密码

    // transient 修饰不可序列化的字段
    private transient Connection connection;
}
```

## transient vs static

| 关键字 | 不参与序列化的原因 |
|--------|------------------|
| `transient` | 显式标记不序列化 |
| `static` | 属于类级别，不属于对象 |

两者反序列化后都不会恢复原来的值，但原因不同：
- `transient` 字段恢复为**默认值**
- `static` 字段保留**当前类的静态值**（因为反序列化不会修改 static 字段）

## 注意事项

1. `transient` 只对 `Serializable` 有效，对 `Externalizable` 无效（后者由开发者控制）
2. `transient` 不能修饰方法（只修饰字段）
3. 可以和 `final` 一起使用（`final transient`）

## 关联知识点
