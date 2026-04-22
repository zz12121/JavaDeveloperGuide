---
title: try-catch-finally 执行顺序面试题
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# try-catch-finally 执行顺序

## Q1：finally 块一定会执行吗？

**A：**
**基本上是的，但有例外**：
- 正常情况：try/catch 不管有无异常，finally **必定执行**
- **例外**：调用 `System.exit()` 会直接终止 JVM，finally 不执行；JVM 崩溃（如 kill -9）也不执行

---

## Q2：try 中有 return，finally 还会执行吗？

**A：**
会执行。执行顺序是：
1. 执行 try 中 return 语句，**暂存返回值**
2. 执行 finally 块
3. 返回暂存的值

示例：
```java
public int test() {
    try {
        return 1;
    } finally {
        System.out.println("finally");  // 先打印这里
    }
}
// 输出：finally，返回 1
```

---

## Q3：finally 中有 return 会怎样？

**A：**
**finally 中的 return 会覆盖 try/catch 中的 return，并且会吞掉异常**：

```java
public int test() {
    try {
        return 1;
    } finally {
        return 2;  // 返回 2，try 的 return 1 被丢弃
    }
}
```

```java
public int test() {
    try {
        throw new RuntimeException();
    } finally {
        return 100;  // 异常被吞掉，返回 100，不抛异常！
    }
}
```

> **结论：不要在 finally 中写 return，这是反模式！**

---

## Q4：finally 中修改 try 的 return 变量，返回值会变吗？

**A：**
**基本类型不会变，引用类型的属性可能会变**：

```java
// 基本类型：不受影响
public int test() {
    int x = 10;
    try {
        return x;   // 暂存 10
    } finally {
        x = 20;     // 修改 x，但返回值已暂存为 10
    }
}
// 返回 10

// 引用类型：影响对象内容
public List<Integer> test() {
    List<Integer> list = new ArrayList<>();
    try {
        return list;   // 暂存 list 引用
    } finally {
        list.add(1);   // 修改对象内容，影响返回结果
    }
}
// 返回 [1]（finally 的修改生效了）
```

---

## Q5：catch 和 finally 哪个先执行？

**A：**
catch 先于 finally 执行。顺序是：`try → catch（如有异常） → finally`。

```java
try {
    throw new Exception();
} catch (Exception e) {
    System.out.println("catch");   // 先
} finally {
    System.out.println("finally"); // 后
}
// 输出：catch → finally
```

---

## 关联知识点

