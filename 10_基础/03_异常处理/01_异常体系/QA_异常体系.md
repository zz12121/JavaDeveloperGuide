---
title: 异常体系
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-21
---

# 异常体系

## Q1：Java 异常体系的整体结构是什么？

**A**：以 `Throwable` 为根，分为两大分支：

```
Throwable
├── Error              （JVM/系统级严重错误，程序无法恢复）
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception          （程序可处理的异常）
    ├── IOException    ← 受检异常（Checked）
    ├── SQLException   ← 受检异常（Checked）
    └── RuntimeException  ← 非受检异常（Unchecked）
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        └── ClassCastException ...
```

**记忆口诀**：Throwable 生两子（Error/Exception），Exception 再分受检（Checked）和非受检（RuntimeException 分支）。

---

## Q2：Error 和 Exception 的区别？

**A**：

| 对比维度 | Error | Exception |
|---------|-------|-----------|
| 含义 | JVM/系统级不可恢复错误 | 程序逻辑/运行时可处理异常 |
| 是否可恢复 | 通常不可恢复 | 大多数可捕获处理 |
| 是否需捕获 | 不需要也无意义 | 受检异常必须处理 |
| 典型代表 | `OutOfMemoryError`、`StackOverflowError` | `IOException`、`NullPointerException` |

**注意**：Error 和 Exception 是**并列关系**，Error 不是 Exception 的子类。`catch(Exception e)` **无法捕获 Error**，只有 `catch(Throwable t)` 才能全部捕获。

---

## Q3：受检异常和非受检异常的区别？

**A**：

| 对比维度 | 受检异常（Checked） | 非受检异常（Unchecked） |
|---------|------------------|----------------------|
| 定义 | Exception 子类（除 RuntimeException 分支） | RuntimeException 及其子类 |
| 编译器要求 | 必须 try-catch 或 throws 声明，否则**编译报错** | 不强制要求，编译通过 |
| 触发场景 | 外部资源（文件、网络、数据库）操作 | 编程错误（空指针、越界、类型错误） |
| 设计意图 | 提醒调用者"此处可能出错，请处理" | 应通过修复代码逻辑来消除 |
| 典型代表 | `IOException`、`SQLException` | `NullPointerException`、`ClassCastException` |

---

## Q4：`catch(Exception e)` 能捕获所有异常吗？

**A**：**不能**。`catch(Exception e)` 只能捕获 `Exception` 及其子类（包括受检异常和 RuntimeException），**无法捕获 Error**（如 `OutOfMemoryError`、`StackOverflowError`）。  
若需要捕获所有可抛出对象（包括 Error），必须使用 `catch(Throwable t)`。  
但**不建议捕获 Error**，因为 JVM 遇到 Error 通常已处于不可恢复状态，强行捕获可能掩盖严重问题。

---

## Q5：子类重写方法时，throws 的范围有限制吗？

**A**：有严格限制，**只针对受检异常**：

- 子类重写方法**只能抛出相同或更小范围的受检异常**，不能抛出更宽泛的受检异常
- 子类重写方法**可以不抛出任何受检异常**（缩小范围）
- **非受检异常（RuntimeException）不受此限制**，可以随意抛出

```java
class Base {
    public void read() throws IOException { }
}
class Sub extends Base {
    @Override
    public void read() throws FileNotFoundException { }  // ✅ FileNotFoundException 是 IOException 子类
    // public void read() throws Exception { }            // ❌ Exception 比 IOException 范围更大
    // public void read() { }                             // ✅ 不抛出也可以
}
```

**原因**：里氏替换原则——子类对象可以替换父类对象，调用者只 catch 了 `IOException`，子类不能突然抛出更大的异常让调用者措手不及。

---

## Q6：finally 一定会执行吗？

**A**：**绝大多数情况会执行**，但有例外：
- `System.exit()` 被调用时，JVM 直接退出，finally **不执行**
- JVM 进程被强制杀死（kill -9）时，finally **不执行**
- 线程被 `stop()` 强制终止时，finally **可能不执行**（已废弃方法）
- 程序所在线程死循环或死锁，不能到达 finally

**有 return 时 finally 仍然执行**：
```java
public int test() {
    try {
        return 1;    // 先保存返回值 1
    } finally {
        return 2;    // finally 中的 return 会覆盖 try 中的 return
    }
}
// 结果：返回 2（强烈不推荐在 finally 中写 return）
```

---

## Q7：什么是异常链？为什么要保留异常链？

**A**：**异常链**是指在捕获一个异常后，将其作为 `cause` 传入新异常中抛出，从而保留完整的异常传播路径。

```java
try {
    // 底层操作，抛出 SQLException
    db.query(sql);
} catch (SQLException e) {
    // 包装成业务异常，但保留原始 cause
    throw new ServiceException("查询用户失败", e);  // ✅ 保留了 e
    // throw new ServiceException("查询用户失败");   // ❌ 丢失了根本原因
}
```

**为什么重要**：
- 保留异常链可以通过 `e.getCause()` 追溯到**根本原因**
- 丢失 cause 会导致排查问题时只看到"包装异常"，无法定位底层真正出错的地方
- 日志框架（如 Logback）会自动打印完整的 `caused by` 链

---

## Q8：如何正确设计自定义异常？

**A**：业务系统中自定义异常的标准做法：

```java
// 推荐：继承 RuntimeException（避免强迫调用方 try-catch）
public class BizException extends RuntimeException {
    private final int code;

    public BizException(int code, String message) {
        super(message);
        this.code = code;
    }

    // 必须提供带 cause 的构造器，保留异常链
    public BizException(int code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public int getCode() { return code; }
}
```

**设计原则**：
1. **业务异常继承 RuntimeException**（Spring、MyBatis 等主流框架都遵循此规范）
2. **必须提供带 `Throwable cause` 的构造器**，保留异常链
3. **提供有意义的错误码和消息**，方便统一异常处理
4. **命名要表达业务语义**：如 `UserNotFoundException`，而非 `MyException`

---

## Q9：NullPointerException 如何避免？Java 14 有什么改进？

**A**：

**避免 NPE 的常见手段**：
```java
// 1. 判空后再使用
if (user != null) { user.getName(); }

// 2. Optional 包装（Java 8+）
Optional.ofNullable(user)
        .map(User::getName)
        .orElse("未知");

// 3. Objects.requireNonNull（快速失败，提前暴露问题）
public void setUser(User user) {
    this.user = Objects.requireNonNull(user, "user 不能为 null");
}
```

**Java 14+ 改进**：引入"有帮助的 NullPointerException"（Helpful NPE），异常信息会精确指出**哪个变量/方法调用链**导致了空指针，不再只显示行号：
```
Cannot invoke "String.length()" because "str" is null
```

---

## Q10：RuntimeException 的常见子类及触发场景？

**A**：

| 异常类 | 触发场景 | 典型代码 |
|--------|---------|---------|
| `NullPointerException` | 对 null 调用方法/访问字段 | `null.length()` |
| `ArrayIndexOutOfBoundsException` | 数组下标越界 | `arr[arr.length]` |
| `ClassCastException` | 强转类型不匹配 | `(String) new Integer(1)` |
| `NumberFormatException` | 字符串转数字格式错误 | `Integer.parseInt("abc")` |
| `ArithmeticException` | 整数除零 | `10 / 0` |
| `IllegalArgumentException` | 方法参数不合法 | `new ArrayList(-1)` |
| `IllegalStateException` | 对象状态不允许此操作 | `iterator.remove()` 未调用 next |
| `UnsupportedOperationException` | 调用了不支持的操作 | 不可修改集合调用 `add()` |
| `ConcurrentModificationException` | 遍历时修改集合 | for-each 中 `list.remove()` |
| `StackOverflowError` | 递归过深（注意：这是 Error） | 无终止条件的递归 |

---

## 关联知识点

- [[try-catch-finally]] - 异常捕获与资源清理机制
- [[try-with-resources]] - 自动关闭资源的语法糖
- [[throws与throw]] - 声明与主动抛出异常
- [[异常链]] - getCause() 与异常包装原则
- [[自定义异常]] - 业务异常设计规范
