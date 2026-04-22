---
title: try-catch-finally 执行顺序
tags:
  - Java/异常处理
  - 原理型
module: 03_异常处理
created: 2026-04-18
---

# try-catch-finally 执行顺序

## 基本规则

```java
try {
    // 1. 先执行 try 块
    // 若抛出异常 → 跳到匹配的 catch
    // 若无异常   → 跳过所有 catch
} catch (ExceptionType e) {
    // 2. 捕获到匹配异常时执行
} finally {
    // 3. 无论是否异常，finally 必执行
    //    （除非 System.exit() 或 JVM 崩溃）
}
```

## 关键执行顺序

### 情况一：无异常
```
try 块执行 → finally 块执行 → 继续后续代码
```

### 情况二：有异常且被捕获
```
try 块抛异常 → 跳出 try → catch 块执行 → finally 块执行 → 继续后续代码
```

### 情况三：有异常但未捕获
```
try 块抛异常 → 无匹配 catch → finally 块执行 → 异常向上抛出（后续代码不执行）
```

### 情况四：finally 中有 return（危险！）

```java
public int test() {
    try {
        return 1;
    } finally {
        return 2;  // ⚠️ 覆盖 try 的 return，最终返回 2
    }
}
```

> **结论：finally 中的 return 会覆盖 try/catch 中的 return，且会吞掉异常！**

### 情况五：catch 中再次抛异常

```java
try {
    // 抛出 A
} catch (Exception e) {
    throw new RuntimeException("包装", e);  // 抛出 B
} finally {
    // 仍然会执行，然后异常 B 向上传播
}
```

## 特殊场景：System.exit()

```java
try {
    System.exit(0);  // JVM 直接退出
} finally {
    // 不会执行！System.exit() 终止 JVM
}
```

## 返回值细节

```java
public int test() {
    int x = 10;
    try {
        x = 20;
        return x;       // 暂存返回值 20
    } finally {
        x = 30;         // 修改 x，但返回值已暂存，不影响结果
        // 最终返回 20（不是 30），除非 finally 中有 return
    }
}
```

> **原理**：return 执行时会将值暂存，finally 修改变量不影响已暂存的返回值。

## 关联知识点

