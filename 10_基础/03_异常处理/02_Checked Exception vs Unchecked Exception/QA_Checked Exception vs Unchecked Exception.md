---
title: Checked vs Unchecked Exception
tags:
  - Java/异常处理
  - 对比型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# Checked vs Unchecked Exception

## Q1：什么是 Checked Exception 和 Unchecked Exception？区别是什么？

**A：**
- **Checked Exception（受检异常）**：继承自 `Exception`（但非 `RuntimeException`），编译期强制要求处理（try-catch 或 throws）。代表外部环境问题，调用者需要感知。
- **Unchecked Exception（非受检异常）**：继承自 `RuntimeException`，编译期不强制处理，代表程序逻辑缺陷。
核心区别：是否在**编译期**强制处理。

---

## Q2：InterruptedException 是 Checked 还是 Unchecked？

**A：**
`InterruptedException` 是 **Checked Exception**，继承自 `Exception`（非 RuntimeException），因此必须显式处理。
这是一道常见陷阱题，很多人误以为线程相关的异常都是 Unchecked。

---

## Q3：为什么 Spring 把 SQLException 封装成了 RuntimeException？

**A：**
Spring 的 `DataAccessException` 体系将 `SQLException`（Checked）统一封装为 `RuntimeException`（Unchecked），理由是：
1. `SQLException` 是底层数据库错误，业务层通常无法恢复，捕获也无意义
2. 强制 try-catch 会导致大量样板代码，降低业务代码可读性
3. 让调用者**自己决定是否处理**，而不是编译器强制，更灵活
这是"Checked Exception 的反例"，说明 Checked 并不总是好设计。

---

## Q4：自定义异常应该继承 Exception 还是 RuntimeException？

**A：**
取决于使用场景：
- 继承 `RuntimeException`（推荐）：现代框架主流做法，调用方自由选择是否处理，减少样板代码
- 继承 `Exception`：需要强制调用方感知并处理时使用（如自定义的 `BizException` 需要上层必须处理）

> 实际项目中，业务异常通常继承 `RuntimeException`，如 `BusinessException extends RuntimeException`。

---

## 关联知识点
