---
title: JDK 17新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Sealed Classes 密封类

## Q1：什么是 Sealed Classes？解决什么问题？

**A：** Sealed Classes（密封类）是 JDK 17 正式引入的特性，通过 `sealed` 关键字限制类的继承层次结构。使用 `permits` 关键字明确列出所有允许的子类，子类必须声明为 `final`、`sealed` 或 `non-sealed` 三种之一。

**解决的问题：**
- 传统 Java 只有 `final`（完全封闭）和无限制继承两个极端，缺少中间状态
- 在需要"允许特定类继承，禁止其他类继承"的场景下（如 SDK 设计、领域模型建模），密封类提供了语言级别的支持，无需通过包私有构造器等 hack 实现

**使用示例：**
```java
public sealed class Shape permits Circle, Rectangle, Square { }

public final class Circle extends Shape { }        // 终止继承
public sealed class Rectangle extends Shape       // 继续密封
        permits FilledRectangle { }
public non-sealed class Square extends Shape { }  // 重新开放
```

---

## Q2：`final`、`sealed`、`non-sealed` 三种子类修饰符有什么区别？

**A：**

| 修饰符 | 含义 | 使用场景 |
|--------|------|---------|
| `final` | 不能再被继承 | 继承链的叶子节点 |
| `sealed` | 继续密封，需再次 `permits` | 中间节点，分层控制继承 |
| `non-sealed` | 恢复为普通类，开放继承 | 控制直接子类，但不限制间接子类 |

三者必须选其一，子类不加任何修饰符会编译报错。

---

## Q3：Sealed Classes 与 enum 有什么区别？

**A：**

| 维度 | enum | sealed class |
|------|------|-------------|
| 实例数量 | 每个枚举值是单例，数量固定 | 子类可以有多个实例 |
| 状态 | 通常无状态或简单状态 | 子类可以有复杂状态和行为 |
| 继承 | 枚举值不可继承 | 子类可继续被继承（取决于修饰符） |
| 适用场景 | 有限常量集合 | 有限但可扩展的类型层次 |

简单说：**enum 是"单例集合"，sealed class 是"受控继承层次"**。当子类需要携带不同的状态时，sealed class 更合适。

---

## Q4：Sealed Classes 与 Records 如何配合使用？

**A：** 这是 JDK 17 最经典的组合，用于定义**代数数据类型（ADT）**：
```java
public sealed interface Expr permits Num, Add, Mul {
    record Num(int value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
}

// Switch 表达式自动穷举，无需 default
int eval(Expr expr) {
    return switch (expr) {
        case Num(var v) -> v;
        case Add(var l, var r) -> eval(l) + eval(r);
        case Mul(var l, var r) -> eval(l) * eval(r);
    };
}
```

**核心价值：** 密封类保证子类有限且已知，Switch 表达式保证穷举检查，Record 保证数据不可变。三者结合实现了**类型安全的模式匹配**。

---

## Q5：Sealed Classes 有哪些编译器约束？

**A：**
1. **子类位置约束：** 模块化项目中 sealed 类与所有 permits 子类必须在同一 module；非模块化项目中必须在同一编译单元（默认同一包）
2. **permits 完整性：** 所有直接子类都必须出现在 permits 列表中，否则编译报错
3. **子类必须声明修饰符：** `final`、`sealed`、`non-sealed` 三选一，不可省略
4. **sealed 类本身不能是匿名类或局部类**

---

## Q6：实际开发中什么时候该用 Sealed Classes？

**A：** 典型使用场景：

1. **领域模型建模：** 支付结果（成功/失败/处理中）、订单状态等有限状态集合
2. **SDK/框架 API 设计：** 限制插件扩展点，只允许特定的实现类
3. **AST（抽象语法树）：** 编译器/解析器中不同节点类型的表示
4. **与 Switch 表达式配合：** 需要编译器保证穷举的类型匹配场景
**选择原则：** 当你发现自己在用注释写"只允许以下几种类型"时，就该考虑用密封类了。
