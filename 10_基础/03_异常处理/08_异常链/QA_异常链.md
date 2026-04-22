---
title: 异常链面试题
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# 异常链

## Q1：什么是异常链？为什么要用异常链？

**A：**
**异常链**是指捕获一个异常后，将其作为 `cause` 包装到新异常中一起抛出，形成层层关联的异常传播路径。
使用异常链的原因：
1. **保留原始信息**：不会丢失底层异常的堆栈和描述
2. **分层解耦**：上层不暴露底层实现细节（SQL/网络异常），但保留调试信息
3. **便于排查**：日志中可以看到完整的调用链路

---

## Q2：如何实现异常链？

**A：**
```java
// 推荐方式：构造方法传入 cause
throw new ServiceException("服务层描述", originalException);

// 也可以：initCause()（只能调一次）
RuntimeException e = new RuntimeException("描述");
e.initCause(originalException);
throw e;
```

关键是自定义异常要提供 `(String message, Throwable cause)` 构造方法：
```java
public class BusinessException extends RuntimeException {
    public BusinessException(String message, Throwable cause) {
        super(message, cause);  // 调用父类，传入 cause
    }
}
```

---

## Q3：getCause() 和 getSuppressed() 有什么区别？

**A：**

| | `getCause()` | `getSuppressed()` |
|--|-------------|------------------|
| 含义 | 引发当前异常的原始原因 | 被抑制的异常（try-with-resources） |
| 数量 | 1 个（或 null） | 0 到多个 |
| 场景 | 异常链包装 | try-with-resources 关闭时抛出的异常 |

---

## Q4：catch 捕获异常后直接 throw e 和包装后 throw 有什么区别？

**A：**
- `throw e`：重新抛出原异常，保留原始堆栈信息，适合不需要转型时
- `throw new XxxException("描述", e)`：包装为新异常，添加上下文信息，适合分层转换

**注意陷阱**：
```java
catch (Exception e) {
    throw new RuntimeException(e.getMessage());  // ❌ 丢失了 cause 和堆栈！
    throw new RuntimeException("描述", e);        // ✅ 保留了完整链路
}
```

---

## Q5：如何打印完整的异常链信息？

**A：**
`e.printStackTrace()` 会打印完整的异常链，包含所有 `caused by` 信息。
也可以用日志框架：
```java
log.error("操作失败", e);  // Slf4j 会自动展开异常链
```

遍历异常链：
```java
Throwable t = e;
while (t != null) {
    System.out.println(t.getClass().getName() + ": " + t.getMessage());
    t = t.getCause();
}
```
