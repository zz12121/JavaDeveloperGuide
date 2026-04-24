---
title: JDK 9新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口私有方法

## 核心结论

JDK 9 允许接口中定义 **private 方法**（包括 `private` 实例方法和 `private static` 方法），用于在接口内部实现代码复用，避免 default 方法之间出现重复代码。private 方法**不会暴露给实现类**，也不能被继承。

---

## 深度解析

### 1. 为什么需要接口私有方法？

JDK 8 引入 default 方法后，如果多个 default 方法有重复逻辑，需要一个地方放公共代码。但不能用 default 方法（会暴露给实现类），也不能用 static 方法（无法访问实例上下文）。JDK 9 的 private 方法解决了这个问题。

```java
public interface Logging {
    // 两个 default 方法有重复逻辑
    default void logInfo(String msg) {
        formatAndLog("INFO", msg);  // 重复
    }

    default void logError(String msg) {
        formatAndLog("ERROR", msg); // 重复
    }

    // JDK 9 之前：重复代码无法提取
    // JDK 9 之后：提取为 private 方法
    private void formatAndLog(String level, String msg) {
        System.out.println("[%s] %s - %s".formatted(
            LocalDateTime.now(), level, msg));
    }
}
```

### 2. 语法规则

```java
public interface MyInterface {
    // private 实例方法：可被 default 方法调用
    private void instanceHelper() {
        System.out.println("私有实例方法");
    }

    // private static 方法：可被 default 和 static 方法调用
    private static void staticHelper() {
        System.out.println("私有静态方法");
    }

    default void method1() {
        instanceHelper(); // ✅
        staticHelper();   // ✅
    }

    static void method2() {
        staticHelper();   // ✅
        // instanceHelper(); // ❌ 静态方法不能调用实例方法
    }
}
```

### 3. 关键规则

| 规则 | 说明 |
|------|------|
| 访问修饰符 | 只能是 `private`，不能用 `protected` 或 `public` |
| 调用范围 | 仅接口内部可调用 |
| 实现类访问 | ❌ 不可访问，不可重写 |
| private 实例方法 | 可被 default 方法和 private 实例方法调用 |
| private static 方法 | 可被 default 方法、static 方法、private 方法调用 |

### 4. 接口方法访问级别演进

| JDK 版本 | 接口支持的方法类型 |
|----------|-------------------|
| JDK 7 及之前 | 只有 `public abstract` 方法 |
| JDK 8 | 新增 `default` 和 `public static` 方法 |
| JDK 9 | 新增 `private` 和 `private static` 方法 |

---

## 代码示例

```java
public interface Validator {
    // public 接口方法
    boolean validate(String input);

    // default 方法
    default void validateAndPrint(String input) {
        boolean result = doValidate(input);
        System.out.println(result ? "验证通过" : "验证失败");
    }

    default void validateOrFail(String input) {
        if (!doValidate(input)) {
            throw new IllegalArgumentException("验证失败: " + input);
        }
    }

    // private 方法：提取公共验证逻辑
    private boolean doValidate(String input) {
        return input != null && !input.isBlank() && input.length() <= 100;
    }
}
```

---


# 集合工厂方法

## 核心结论

JDK 9 为 `List`、`Set`、`Map` 接口新增了 `of()` 系列工厂方法，用于快速创建**不可变集合**。相比 `Arrays.asList()` 和 `Collections.unmodifiableList()`，`of()` 更简洁且创建的集合**天然不可变**（null 值、增删操作都会抛异常）。

---

## 深度解析

### 1. 基本用法

```java
// List.of()
var list1 = List.of("A", "B", "C");
var emptyList = List.of();

// Set.of()
var set1 = Set.of("A", "B", "C");
var emptySet = Set.of();

// Map.of()
var map1 = Map.of("key1", "value1", "key2", "value2");
var emptyMap = Map.of();

// Map.ofEntries() — 适合更多键值对
var map2 = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3)
);
```

### 2. 不可变性

```java
List<String> list = List.of("A", "B", "C");
list.add("D");      // UnsupportedOperationException
list.set(0, "X");   // UnsupportedOperationException
list.remove(0);     // UnsupportedOperationException
```

> 创建后**不可增删改**，任何修改操作都抛 `UnsupportedOperationException`。

### 3. null 值限制

```java
List.of("A", null, "C");  // NullPointerException
Set.of(null);             // NullPointerException
Map.of("key", null);      // NullPointerException
```

> `of()` **不允许 null 元素**。如需允许 null，使用 `ArrayList` 或 `Collections.singletonList()`。

### 4. Set 的去重

```java
Set.of("A", "B", "A");  // IllegalArgumentException: 重复元素
```

> `Set.of()` 会在创建时检查重复，发现重复立即抛 `IllegalArgumentException`。

### 5. 与 Arrays.asList() 对比

| 对比项 | `Arrays.asList()` | `List.of()` |
|--------|-------------------|-------------|
| 可变性 | 固定大小，但元素可 set | 完全不可变（元素也不可 set） |
| null 元素 | 允许 | 不允许 |
| null 数组参数 | 允许 | 不允许 |
| JDK 版本 | JDK 1.2 | JDK 9 |
| 与原数组关系 | 共享底层数组 | 独立，不受影响 |

### 6. 实现原理

```java
// 不同元素数量使用不同实现类，避免数组开销
List.of()                  // ListN.EMPTY（空实例）
List.of("A")              // ListN（1个元素）
List.of("A", "B", "C")    // ListN（N个元素，内部用数组存储）
```

> `List.of()` 返回的是不可变的 `ListN` 实例，实现了序列化等接口。

---

## 代码示例

```java
// JDK 9 之前
List<String> names = Collections.unmodifiableList(
    Arrays.asList("Alice", "Bob", "Charlie")
);

// JDK 9 之后
List<String> names = List.of("Alice", "Bob", "Charlie");

// 常见使用场景：方法返回常量集合
public List<String> getSupportedFormats() {
    return List.of("JSON", "XML", "YAML");
}
```

---

## 关联知识点
