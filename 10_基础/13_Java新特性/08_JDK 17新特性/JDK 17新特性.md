---
title: JDK 17新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Sealed Classes 密封类（JDK 15 预览/17 正式）

## 核心结论

Sealed Classes（密封类）通过 `sealed` 关键字限制哪些类可以继承或实现它，配合 `permits` 指定允许的子类。子类必须使用 `final`（不可再继承）、`sealed`（继续密封）或 `non-sealed`（开放继承）三个修饰符之一。目的是**在继承的灵活性与封装的安全性之间取得平衡**。

## 深度解析

### 1. 背景与动机

传统 Java 中类的继承控制只有两个极端：
- **无限制继承**（默认）：任何类都可继承，无法控制
- **禁止继承**（`final`）：完全封闭，无法扩展

实际开发中经常需要「允许某些类继承，但不允许其他类继承」的中间状态。在 JDK 17 之前只能靠 `protected` 构造器 + 同包限制等 hack 手段实现，密封类提供了语言级别的支持。

### 2. 基本语法

```java
// 密封类：指定只允许 Circle、Rectangle、Square 三个子类继承
public sealed class Shape permits Circle, Rectangle, Square {
    // ...
}

// 子类必须三选一：final / sealed / non-sealed
public final class Circle extends Shape { }       // 终止继承

public sealed class Rectangle extends Shape       // 继续密封
        permits FilledRectangle {
    // ...
}

public non-sealed class Square extends Shape { }  // 重新开放继承
```

### 3. 三种子类修饰符

| 修饰符 | 含义 | 使用场景 |
|--------|------|---------|
| `final` | 不能再被继承 | 叶子节点，继承链终止 |
| `sealed` | 继续密封，需再次 permits | 中间节点，限制下一层扩展 |
| `non-sealed` | 恢复为普通类，开放继承 | 需要控制直接子类，但不限制间接子类 |

### 4. 同样适用于接口

```java
public sealed interface Service permits DatabaseService, CacheService, LoggingService {
    void execute();
}

public final class DatabaseService implements Service {
    @Override public void execute() { /* ... */ }
}

public non-sealed class CacheService implements Service {
    @Override public void execute() { /* ... */ }
    // 可被任意类继承
}
```

### 5. 编译器约束规则

- **子类必须在同一模块中**（模块化项目）：sealed 类和所有 permits 子类必须在同一 module 中
- **子类必须在同一包中**（非模块化项目）：默认要求同一包
- **`permits` 列表必须完整**：所有直接子类都必须在 permits 中声明，否则编译报错
- **子类必须声明修饰符**：`final`、`sealed`、`non-sealed` 三选一，缺一不可

### 6. 与 Records 的天然配合

密封类 + Record 是 JDK 17 的经典组合，非常适合定义受限的代数数据类型（ADT）：

```java
public sealed interface Expr permits Num, Add, Mul {
    record Num(int value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
}

// 使用时 match 所有子类，编译器可检查是否穷举
int eval(Expr expr) {
    return switch (expr) {
        case Num(var v) -> v;
        case Add(var l, var r) -> eval(l) + eval(r);
        case Mul(var l, var r) -> eval(l) * eval(r);
        // 无需 default，编译器保证穷举
    };
}
```

> **这是最重要的面试场景**：密封类 + Switch 表达式实现类型安全的模式匹配，编译器自动检查穷举性。

### 7. 与 enum 的对比

| 对比维度 | enum | sealed class/interface |
|---------|------|----------------------|
| 实例数量 | 编译时确定，每个枚举值是单例 | 每个子类可以有多个实例 |
| 继承 | 枚举值不能被继承 | sealed 子类可被继承（取决于修饰符） |
| 状态 | 枚举值通常无状态或固定状态 | 子类可以有复杂的状态和行为 |
| 适用场景 | 有限集合的固定常量 | 有限但可扩展的类型层次结构 |

## 代码示例

```java
// 支付结果建模
public sealed class PaymentResult
        permits PaymentSuccess, PaymentFailure, PaymentPending {

    private final String transactionId;

    protected PaymentResult(String transactionId) {
        this.transactionId = transactionId;
    }

    public String getTransactionId() { return transactionId; }
}

public final class PaymentSuccess extends PaymentResult {
    private final BigDecimal amount;

    public PaymentSuccess(String txId, BigDecimal amount) {
        super(txId);
        this.amount = amount;
    }

    public BigDecimal getAmount() { return amount; }
}

public final class PaymentFailure extends PaymentResult {
    private final String errorCode;
    private final String errorMessage;

    public PaymentFailure(String txId, String code, String message) {
        super(txId);
        this.errorCode = code;
        this.errorMessage = message;
    }
}

public final class PaymentPending extends PaymentResult {
    private final LocalDateTime estimatedTime;

    public PaymentPending(String txId, LocalDateTime estimatedTime) {
        super(txId);
        this.estimatedTime = estimatedTime;
    }
}

// 模式匹配，编译器保证穷举
String handlePayment(PaymentResult result) {
    return switch (result) {
        case PaymentSuccess s -> "支付成功：" + s.getAmount();
        case PaymentFailure f -> "支付失败：" + f.getErrorMessage();
        case PaymentPending p -> "支付处理中，预计完成：" + p.getEstimatedTime();
    };
}
```

## 关联知识点
