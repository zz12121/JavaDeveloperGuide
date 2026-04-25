---
title: JDK 10新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# var 局部变量类型推断

## 核心结论

`var` 是 JDK 10 引入的**局部变量类型推断**关键字，编译器根据右侧赋值表达式自动推断变量类型。`var` 仅用于**局部变量**，不能用于成员变量、方法参数、返回值。编译后字节码与显式声明类型完全一致，**不影响运行时性能**。

---

## 深度解析

### 1. 基本用法

```java
// 传统写法
ArrayList<String> list = new ArrayList<String>();
HashMap<String, Integer> map = new HashMap<String, Integer>();

// var 写法（编译器自动推断）
var list = new ArrayList<String>();     // 推断为 ArrayList<String>
var map = new HashMap<String, Integer>(); // 推断为 HashMap<String, Integer>
var stream = list.stream();             // 推断为 Stream<String>
var entry = map.entrySet().iterator().next(); // 推断为 Map.Entry<String, Integer>
```

### 2. 适用场景

```java
// ✅ 局部变量声明并初始化
var name = "hello";                    // String
var num = 42;                          // int
var list = List.of("A", "B", "C");     // List<String>

// ✅ try-with-resources
try (var reader = new BufferedReader(new FileReader("file.txt"))) {
    // 自动推断为 BufferedReader
}

// ✅ for 循环
for (var item : list) {
    System.out.println(item);           // 推断为 String
}
```

### 3. 不适用场景

```java
// ❌ 未初始化
var x;                    // 编译错误：无法推断

// ❌ 赋值为 null
var y = null;             // 编译错误：无法推断类型

// ❌ 成员变量
public var field = 10;    // 编译错误

// ❌ 方法参数
public void method(var param) {}  // 编译错误

// ❌ 方法返回值
public var method() { return 1; }  // 编译错误

// ❌ Lambda 表达式
var func = s -> s.length();  // 编译错误：无法推断目标类型

// ❌ 数组初始化
var arr = {1, 2, 3};        // 编译错误
var arr2 = new int[]{1, 2}; // ✅ 可以
```

### 4. 原理

`var` 纯粹是编译期语法糖，**不涉及运行时类型擦除或动态类型**。编译后字节码中变量的类型信息完全保留：

```java
// 源码
var list = new ArrayList<String>();

// 编译后字节码等价于
ArrayList list = new ArrayList();
```

### 5. 最佳实践

| 场景 | 建议 |
|------|------|
| 类型名很长（如 `Map.Entry<String, List<Integer>>`） | ✅ 推荐用 var |
| 右侧类型一目了然（如 `var list = new ArrayList<>();`） | ✅ 推荐 |
| 右侧需要仔细看才能推断类型 | ❌ 不推荐，降低可读性 |
| Diamond 退化问题 | var 避免 `List<String> list = new ArrayList<>();` 无法推断泛型的问题 |

### 6. 常见陷阱

```java
// 陷阱：var 推断为基本类型而非包装类
var a = 1;         // int（不是 Integer）
var b = 1.0;       // double（不是 BigDecimal）

// 陷阱：泛型推断可能不如预期
var list = new ArrayList<>();  // 推断为 ArrayList<Object>，不是 ArrayList<String>
// 应该写：
var list = new ArrayList<String>();  // 明确指定泛型
```

---

---

### 7. Lambda 表达式中的 var 参数（JDK 11）

JDK 10 的 `var` 不能直接用于 Lambda 参数，但 JDK 11 扩展了 `var` 的使用范围：

```java
// ❌ JDK 10：Lambda 参数不能用 var
Function<String, String> f1 = (var x) -> x.toUpperCase();  // 编译错误

// ✅ JDK 11+：Lambda 参数可以使用 var
Function<String, String> f2 = (var x) -> x.toUpperCase();

// ✅ JDK 11+：配合注解使用（这是 var 在 Lambda 中的主要用途）
Function<String, String> f3 = (@NonNull var x) -> x.toUpperCase();
// 用于在 Lambda 表达式中为参数添加注解（之前做不到）

@Retention(RetentionPolicy.RUNTIME)
@interface NonNull {}
```

> **实际用途**：Lambda 参数加注解是 `var` 在 Lambda 中使用 var 的唯一实际意义——因为 Lambda 参数的类型可以从上下文推断，加 `var` 本身没意义，但如果需要注解（用于 null 检查、校验等），就必须用 `var` 显式声明参数类型。

---

## 关联知识点


