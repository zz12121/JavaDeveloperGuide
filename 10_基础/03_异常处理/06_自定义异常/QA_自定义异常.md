---
title: 自定义异常
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# 自定义异常

## Q1：如何创建自定义异常？

**A：**
继承 `RuntimeException`（Unchecked，推荐）或 `Exception`（Checked），通常需要：
1. 定义有意义的类名（如 `UserNotFoundException`）
2. 提供多个构造方法（message、message+cause）
3. 可扩展错误码、业务字段

```java
public class BusinessException extends RuntimeException {
    private final int code;
    
    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }
    
    public BusinessException(int code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }
}
```

---

## Q2：自定义异常应该继承 RuntimeException 还是 Exception？

**A：**
大多数场景推荐继承 **RuntimeException**：
- 调用方无需强制 try-catch，减少样板代码
- 现代框架（Spring）主流做法
- 在全局异常处理器统一处理，不需要层层 throws 声明

继承 `Exception` 的场景：
- 需要强制调用方感知并处理时（如框架 API 设计）

---

## Q3：项目中如何设计异常体系？

**A：**
常见分层设计：

```
RuntimeException
└── BaseException（基础异常，含 code + message）
    ├── BusinessException（业务异常，可预期）
    │   ├── UserNotFoundException
    │   ├── OrderNotFoundException
    │   └── PermissionDeniedException
    └── SystemException（系统异常，不可预期）
        ├── RemoteCallException
        └── DatabaseException
```

配合 Spring `@RestControllerAdvice` 做统一异常处理，返回统一格式的错误响应。

---

## Q4：捕获到底层异常后，应该直接抛出还是包装后抛出？

**A：**
**应该包装后抛出**（保留原始异常作为 cause），即异常链模式：

```java
try {
    userDao.findById(id);
} catch (DataAccessException e) {
    // 包装为业务异常，保留原始异常
    throw new UserNotFoundException("用户不存在，id=" + id, e);
}
```

直接 `throw e` 暴露底层实现细节（如 SQL 异常），对调用方不友好，且不利于分层解耦。
