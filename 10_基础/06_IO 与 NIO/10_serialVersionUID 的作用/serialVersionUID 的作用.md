---
title: serialVersionUID 的作用
tags:
  - Java/IO
  - 原理型
module: 06_IO与NIO
created: 2026-04-18
---

# serialVersionUID 的作用

## 核心作用

`serialVersionUID` 是序列化版本的唯一标识符，用于**版本兼容性校验**。

反序列化时，JVM 会比较：
1. 字节流中的 `serialVersionUID`
2. 当前类中的 `serialVersionUID`

不一致 → 抛 `java.io.InvalidClassException`

## 不定义的后果

如果类没有显式定义 `serialVersionUID`，JVM 会根据类结构**自动生成**：

```java
// 自动生成规则（基于类结构：字段、方法、修饰符等）
private static final long serialVersionUID = -6728909276912345678L;
```

**问题**：一旦类结构发生变化（增删字段、修改方法等），自动生成的 UID 也会变化，导致之前序列化的数据无法反序列化。

```java
// 第一次：类有 name 字段，序列化 user.ser
// serialVersionUID = 123456L

// 第二次：类新增了 age 字段
// serialVersionUID = 789012L（变了！）

// 反序列化 → InvalidClassException！
```

## 正确做法：显式定义

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 显式定义

    private String name;
    private int age;
    // 以后新增字段不会影响反序列化
}
```

显式定义后：
- 新增字段：反序列化时新字段为默认值 ✅
- 删除字段：反序列化时忽略多余的字段 ✅
- 修改字段类型：可能报错（建议慎重）

## 版本变化处理策略

| 变化类型 | 是否兼容 | 说明 |
|---------|---------|------|
| 新增字段 | ✅ | 新字段为默认值 |
| 删除字段 | ✅ | 忽略 |
| 字段改名 | ❌ | 类型匹配但名字不对应 |
| 修改字段类型 | ❌ | 可能 ClassCastException |
| 修改类名 | ❌ | 找不到类 |
| 修改修饰符（transient 等） | ❌ | 可能不匹配 |

## 最佳实践

```java
public class User implements Serializable {
    // 1. 始终显式定义
    private static final long serialVersionUID = 1L;

    // 2. 敏感字段加 transient
    private transient String password;

    // 3. 可以递增版本号表示有变化
    // private static final long serialVersionUID = 2L;
}
```

## 关联知识点