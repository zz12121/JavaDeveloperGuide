---
title: Lambda 表达式语法
tags:
  - Java/Lambda
  - 原理型
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# Lambda 表达式语法

## 核心结论

Lambda 表达式是 JDK 8 引入的语法糖，用于简化**函数式接口**的匿名实现。基本语法为 `(参数列表) -> {方法体}`。Lambda 本质上是一个**匿名内部类的简化写法**，编译后由 JVM 通过 `invokedynamic` 生成实现类。

## 深度解析

### 基本语法

```java
// 完整形式
(String s) -> { return s.length(); }

// 省略参数类型（编译器推断）
(s) -> { return s.length(); }

// 单参数省略括号
s -> { return s.length(); }

// 单行方法体省略大括号和 return
s -> s.length()

// 无参数
() -> System.out.println("hello")

// 多参数
(a, b) -> a + b
```

### Lambda 的各种形式

```java
// 1. 无参无返回
Runnable r = () -> System.out.println("running");

// 2. 有参无返回
Consumer<String> c = s -> System.out.println(s);

// 3. 有参有返回
Function<Integer, Integer> f = x -> x * x;

// 4. 多行方法体
Comparator<Integer> comp = (a, b) -> {
    System.out.println("comparing: " + a + ", " + b);
    return a - b;
};

// 5. 方法体中引用外部变量
int factor = 2;
Function<Integer, Integer> multiply = x -> x * factor;
```

### Lambda 的本质原理

Lambda 并非简单的语法糖，其底层实现依赖 `invokedynamic`：

```
源码：Function<Integer, Integer> f = x -> x * x;
    ↓ 编译
字节码：invokedynamic #makeFunction, MethodType(...)
    ↓ 运行时
JVM 调用 LambdaMetafactory
    ↓
动态生成函数式接口的实现类
```

- 不是匿名内部类（不会生成 `.class` 文件）
- 通过 `LambdaMetafactory` 动态生成实现类
- 首次调用后缓存生成的类，后续调用直接复用

### Lambda 的使用场景

```java
// 集合遍历
list.forEach(item -> System.out.println(item));

// 排序
list.sort((a, b) -> a.compareTo(b));

// Stream 操作
list.stream().filter(x -> x > 0).map(x -> x * 2).collect(Collectors.toList());

// 线程
new Thread(() -> doSomething()).start();
```

## 关联知识点

