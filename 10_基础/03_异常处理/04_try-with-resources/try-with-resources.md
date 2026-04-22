---
title: try-with-resources
tags:
  - Java/异常处理
  - 原理型
module: 03_异常处理
created: 2026-04-18
---

# try-with-resources（自动资源关闭）

## 基本语法（JDK 7+）

```java
try (ResourceType resource = new ResourceType()) {
    // 使用资源
} catch (Exception e) {
    // 处理异常
}
// resource.close() 自动调用，无需手动写 finally
```

> 要求：资源类必须实现 `AutoCloseable` 接口（或 `Closeable`）

## 对比：传统 vs try-with-resources

```java
// 传统写法（容易忘记关闭，finally 中还可能抛异常）
InputStream is = null;
try {
    is = new FileInputStream("file.txt");
    // 读取
} finally {
    if (is != null) is.close();  // 可能再次抛异常
}

// try-with-resources（推荐）
try (InputStream is = new FileInputStream("file.txt")) {
    // 读取
}  // 自动关闭，代码更简洁
```

## 多资源写法

```java
// JDK 7：多个资源，关闭顺序与打开顺序相反
try (
    InputStream in = new FileInputStream("input.txt");
    OutputStream out = new FileOutputStream("output.txt")
) {
    // 使用 in 和 out
}
// 关闭顺序：out.close() → in.close()

// JDK 9+：可以在括号外声明资源
final InputStream in = new FileInputStream("input.txt");
try (in) {
    // 使用已有变量
}
```

## 异常抑制（Suppressed Exception）

当 try 块和 close() 都抛出异常时：
- `close()` 的异常被**抑制（suppressed）**，不会覆盖主异常
- 主异常的 `getSuppressed()` 方法可获取被抑制的异常

```java
try (MyResource r = new MyResource()) {
    throw new Exception("主异常");
    // close() 也抛异常 → 被抑制
} catch (Exception e) {
    System.out.println(e.getMessage());            // 主异常
    System.out.println(e.getSuppressed()[0]);       // 被抑制的 close 异常
}
```

> 对比 try-finally：finally 中的异常会**覆盖**主异常（主异常丢失）

## 实现原理

try-with-resources 是语法糖，编译器自动转为：
1. 正常执行后调用 `close()`
2. 发生异常时，先记录主异常，再调用 `close()`，若 close 再抛异常则将其 addSuppressed 到主异常

## 实现细节
### 一、 底层容器：`Throwable`类的源码支持

在 JDK 7 之前，异常一旦抛出，就只是一个孤立的对象。为了支持“一个主异常携带多个被抑制的异常”，JDK 7 在所有的异常的祖先类 **`java.lang.Throwable`​ 中动了刀子。

它主要增加了三个核心部分：
1. **一个存储容器**：`private List<Throwable> suppressedExceptions;`
2. **一个添加方法**：`addSuppressed(Throwable exception)`
3. **一个获取方法**：`getSuppressed()`

我们来看一下 JDK 中 `addSuppressed`的核心源码逻辑（精简版）：
```java
// Throwable.java (JDK 8)
public final synchronized void addSuppressed(Throwable exception) {
    // 1. 防护校验：不能把自己加给自己，也不能加个空值
    if (exception == this)
        throw new IllegalArgumentException("Self-suppression not permitted", exception);
    if (exception == null)
        throw new NullPointerException("Cannot suppress a null exception");

    // 2. 懒加载初始化一个 ArrayList 来存储被抑制的异常
    if (suppressedExceptions == null) 
        return; // 默认情况下不记录被抑制的异常
        
    if (suppressedExceptions == SUPPRESSED_SENTINEL) {
        suppressedExceptions = new ArrayList<>(1);
    }
    
    // 3. 将被抑制的异常加入到列表中
    suppressedExceptions.add(exception);
}
```

**大白话翻译**：JDK 只是在 `Throwable`里塞了一个 `ArrayList`，专门用来当“小弟”异常的收容所。谁调用 `addSuppressed`，就把谁塞进这个名单里。

---
### 二、 幕后黑手：编译器的字节码魔法

`try-with-resources`看起来像是一个高级的语法糖，但 JVM 其实根本不知道它的存在。**这一切都是 `javac`编译器在编译阶段帮你硬核展开的**。
我们通过一个极简的 Demo 来看编译器到底干了什么：
**源码（你写的清爽代码）：**
```java
try (MyResource res = new MyResource()) {
    throw new Exception("Try Block Exception");
} catch (Exception e) {
    System.out.println(e.getMessage());
}
```
**编译后（通过 JD-GUI 等反编译工具看到的字节码逻辑）：**
编译器会把 `try-with-resources`拆解成一个非常严密的 `try-catch-finally`结构：
```java
// 1. 声明变量，并调用 close() 时可能产生的异常变量
MyResource res = null;
Exception var2 = null; // var2 用来承接 try块 产生的主异常

try {
    res = new MyResource();
    throw new Exception("Try Block Exception");
} catch (Exception e) {
    var2 = e; // 捕获 try块 的异常，赋值给 var2
    throw e;  // 重新抛出
} finally {
    // 重点来了：只要 res 不为空，就必须保证调用 close()
    if (res != null) {
        if (var2 != null) { 
            // 如果 try块 有异常，那么 close() 的异常就被抑制
            try {
                res.close();
            } catch (Exception var3) {
                // 核心代码：把 close 的异常附加到主异常上！
                var2.addSuppressed(var3); 
            }
        } else {
            // 如果 try块 没异常，那直接调用 close()，有异常就正常抛
            res.close();
        }
    }
}
```
### 💡 核心逻辑剖析

编译器生成的代码非常严谨，它的处理规则如下：
1. **优先级判定**：它会用一个临时变量（如 `var2`）死盯着 `try`块里有没有异常。
2. **如果 `try`块有异常**：那么在 `finally`里调用 `close()`时，如果 `close()`也炸了，编译器就会自动为你补上一句 `var2.addSuppressed(close的异常)`。这就保证了 **主异常绝对不会被覆盖**。
3. **如果 `try`块没异常**：那 `close()`如果炸了，就直接抛出来，因为此时它就是唯一的异常。
### 总结
所谓的“异常抑制”，拆到底层也就是这么点事儿：
**`Throwable`类提供了一个 `List`作为背包，`javac`编译器在编译时硬编码了 `addSuppressed()`的调用逻辑。**

---

## 关联知识点

