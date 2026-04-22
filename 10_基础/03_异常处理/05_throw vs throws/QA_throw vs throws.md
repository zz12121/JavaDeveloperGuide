---
id: qa_33
title: throw vs throws 面试题
tags:
  - Java/异常处理
  - 对比型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# throw vs throws

## Q1：throw 和 throws 的区别是什么？

**A：**

| | `throw` | `throws` |
|--|---------|----------|
| 位置 | 方法体内 | 方法声明上 |
| 含义 | 主动抛出一个异常实例 | 声明该方法可能抛出的异常类型 |
| 跟随内容 | 异常对象 | 异常类（可多个） |
| 执行时机 | 运行时 | 编译期检查 |

---

## Q2：throw 之后还能写代码吗？

**A：**
不能。`throw` 后的语句属于**不可达代码（unreachable code）**，编译器会报错：

```java
throw new RuntimeException("错误");
System.out.println("这里无法执行");  // ❌ 编译报错：Unreachable statement
```

---

## Q3：子类重写父类方法时，throws 声明有什么限制？

**A：**
子类重写方法时，**throws 的异常范围不能比父类更宽**：
- ✅ 可以不声明（更窄）
- ✅ 可以声明父类异常的子类（更具体）
- ❌ 不能声明父类没有的 Checked Exception（更宽，编译报错）

原因：多态调用时，调用方基于父类签名做 try-catch，如果子类抛了更多受检异常，调用方无法捕获。

---

## Q4：throw 可以抛出 Checked Exception 不用 try-catch 吗？

**A：**
不行。如果方法内 `throw` 了 Checked Exception，则：
- 必须用 `try-catch` 在方法内部处理，或
- 在方法签名上用 `throws` 声明，把责任交给调用方

```java
// 必须二选一
public void test() throws IOException {     // 方式1：声明
    throw new IOException("IO error");
}

public void test() {
    try {
        throw new IOException("IO error");  // 方式2：内部捕获
    } catch (IOException e) { ... }
}
```

---

## Q5：可以在 finally 中 throw 异常吗？会怎样？

**A：**
可以，但不推荐。finally 中 throw 异常会**覆盖 try/catch 中的异常**，导致原始异常丢失：

```java
try {
    throw new RuntimeException("原始异常");
} finally {
    throw new RuntimeException("finally 异常");  // 原始异常被覆盖！
}
// 最终抛出：finally 异常
```

这是不建议在 finally 中 throw 的原因，推荐使用 try-with-resources 代替。

---

## 关联知识点

