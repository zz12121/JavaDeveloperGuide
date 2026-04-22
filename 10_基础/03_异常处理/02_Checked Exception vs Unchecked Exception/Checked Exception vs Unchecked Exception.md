---
title: Checked Exception vs Unchecked Exception
tags:
  - Java/异常处理
  - 对比型
module: 03_异常处理
created: 2026-04-18
---

# Checked Exception vs Unchecked Exception

## 核心区分

| 对比项 | Checked Exception | Unchecked Exception |
|--------|------------------|---------------------|
| 继承关系 | `Exception` 非 `RuntimeException` 子类 | `RuntimeException` 及其子类 |
| 编译期检查 | ✅ 必须 try-catch 或 throws 声明 | ❌ 不强制处理 |
| 典型代表 | `IOException`、`SQLException`、`ClassNotFoundException` | `NullPointerException`、`ClassCastException`、`IllegalArgumentException` |
| 表示含义 | 外部环境异常（文件不存在、网络中断） | 程序逻辑缺陷（空指针、数组越界） |
| 使用场景 | 调用者必须知道并处理 | 调用者无法预期，运行时自动抛出 |

## 常见 Checked Exception

| 类名 | 触发场景 |
|------|----------|
| `IOException` | 文件/流操作失败 |
| `FileNotFoundException` | 文件不存在 |
| `SQLException` | 数据库操作异常 |
| `ClassNotFoundException` | 反射找不到类 |
| `InterruptedException` | 线程中断 |
| `ParseException` | 日期/数字解析失败 |

## 常见 Unchecked Exception

| 类名 | 触发场景 |
|------|----------|
| `NullPointerException` | 对 null 引用调用方法/属性 |
| `ArrayIndexOutOfBoundsException` | 数组下标越界 |
| `ClassCastException` | 类型强制转换失败 |
| `NumberFormatException` | 字符串转数字格式错误 |
| `IllegalArgumentException` | 非法参数 |
| `IllegalStateException` | 状态不合法 |
| `ArithmeticException` | 算术错误（如除以零） |
| `StackOverflowError` | 无限递归（注意：Error） |

## 设计原则

> **什么时候用 Checked？什么时候用 Unchecked？**

- **Checked**：外部资源依赖（IO、网络、DB），调用者需要感知并决策如何处理
- **Unchecked**：程序员的错误（空指针、越界），应该通过防御性编程规避，而不是 catch

> Spring 框架大量使用 `RuntimeException`（如 `DataAccessException`），将受检异常转换为非受检，减少代码噪音。

---

## 关联知识点

