---
title: JDK 14新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Switch 表达式

## 核心结论

JDK 14 正式引入 **Switch 表达式**（JDK 12 预览），使用 `->` 箭头语法替代 `:` 冒号语法，并支持**返回值**。新语法消除了 fall-through 穿透问题，无需 `break`，且支持**多常量匹配**（用逗号分隔）。JDK 17 后可以在 case 中使用模式匹配。

---

## 深度解析

### 1. 旧语法 vs 新语法

```java
// JDK 14 之前：语句形式
String result;
switch (day) {
    case MONDAY:
    case FRIDAY:
        result = "忙碌";
        break;
    case SATURDAY:
    case SUNDAY:
        result = "休息";
        break;
    default:
        result = "普通";
}

// JDK 14+：表达式形式
String result = switch (day) {
    case MONDAY, FRIDAY -> "忙碌";        // 多常量匹配 + 箭头
    case SATURDAY, SUNDAY -> "休息";
    default -> "普通";
};
```

### 2. 关键改进

| 改进 | 旧语法 | 新语法 |
|------|--------|--------|
| 语法 | `case x:` + `break` | `case x ->` |
| 穿透 | 需要 break 防止 | 箭头语法无穿透 |
| 多常量 | 需要 case 堆叠 | 逗号分隔 `case A, B ->` |
| 返回值 | 需要外部变量 | 可直接返回值 |
| 表达式 | 只是语句 | 可以是表达式 |

### 3. case 中使用代码块

```java
String result = switch (day) {
    case MONDAY -> {
        log("周一工作开始");
        yield "忙碌";  // ⚠️ 用 yield 返回值，不是 return
    }
    case SATURDAY -> {
        log("周末休息");
        yield "休息";
    }
    default -> "普通";
};
```

> `yield` 是 JDK 14 新增关键字，用于从 switch 表达式的代码块中返回值。不能用 `return`（return 用于退出方法）。

### 4. 与变量作用域

```java
// 旧语法：整个 switch 共享作用域，变量容易冲突
switch (x) {
    case 1:
        String s = "one";    // 作用域泄漏到其他 case
        break;
    case 2:
        String s = "two";    // ❌ 编译错误：s 已定义
        break;
}

// 新语法：箭头语法每个 case 独立作用域
switch (x) {
    case 1 -> { String s = "one"; yield s; }
    case 2 -> { String s = "two"; yield s; }  // ✅ 不冲突
}
```

### 5. 枚举与 null

```java
// switch 表达式必须穷举所有可能
enum Color { RED, GREEN, BLUE }

String result = switch (color) {
    case RED -> "红";
    case GREEN -> "绿";
    case BLUE -> "蓝";
    // 不需要 default（枚举已穷举）
};

// 注意：switch 表达式对 null 不免疫
Color color = null;
switch (color) { ... }  // ❌ NullPointerException
```

### 6. JDK 17+ 模式匹配（预览）

```java
// JDK 17 开始支持在 switch 中使用 instanceof 模式匹配
static String formatter(Object obj) {
    return switch (obj) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        case String s  -> String.format("String %s", s);
        default        -> obj.toString();
    };
}
```

---


# Pattern Matching for instanceof

## 核心结论

JDK 16 正式引入 **Pattern Matching for instanceof**（JDK 14 预览），允许在 `instanceof` 检查的同时**声明类型变量并自动转型**。消除了显式强转的冗余代码，让代码更简洁安全。

---

## 深度解析

### 1. 旧写法 vs 新写法

```java
// JDK 16 之前：先判断再强转（繁琐，容易出错）
if (obj instanceof String) {
    String s = (String) obj;      // 手动强转
    System.out.println(s.length());
}

// JDK 16+：模式匹配（一步到位）
if (obj instanceof String s) {    // 判断 + 声明 + 自动转型
    System.out.println(s.length()); // s 已经是 String 类型
}
```

### 2. 作用域规则

```java
if (obj instanceof String s) {
    // ✅ s 在此作用域内可用，类型为 String
    System.out.println(s.length());
}
// ❌ s 在此作用域外不可用

// 嵌套使用
if (obj instanceof String s && s.length() > 5) {
    // ✅ s 在 && 右侧也可用
}

// 注意：|| 运算符中 s 不可用
if (obj instanceof String s || s.length() > 5) {
    // ❌ 编译错误：s 可能未初始化
}
```
> 在 `&&` 中，由于短路求值，`instanceof` 为 true 后才会执行右侧，所以 s 安全可用。但 `||` 左侧为 true 时右侧不执行，s 可能未初始化。

### 3. 与 else 结合

```java
if (obj instanceof String s) {
    System.out.println("字符串: " + s);
} else if (obj instanceof Integer i) {
    System.out.println("整数: " + i);
} else {
    System.out.println("其他: " + obj);
}
```

### 4. 消除 Liskov 违反代码

```java
// 旧写法中常见的不安全模式
if (obj instanceof String) {
    // 假设 obj 是 String，但强转可能不安全
    process((String) obj);
}

// 新写法更安全，编译器保证类型
if (obj instanceof String s) {
    process(s);  // s 类型安全
}
```

### 5. 模式变量特性

|特性|说明|
|---|---|
|声明方式|`instanceof Type variableName`|
|作用域|仅在条件为 true 的分支内有效|
|类型|编译器保证，不会出错|
|是否 final|模式变量隐式为 final，不能重新赋值|
```java
if (obj instanceof String s) {
    s = "new value";  // ❌ 编译错误：s 是 final
}
```

### 6. 配合 Records 使用（JDK 16+）

```java
record Point(int x, int y) {}

// 先判断类型，再解构
if (obj instanceof Point(int x, int y)) {  // JDK 19+ 解构模式
    System.out.println("x=" + x + ", y=" + y);
}
```

---

## 代码示例

```java
// 实际场景：equals 方法中的类型检查
@Override
public boolean equals(Object obj) {
    // 旧写法
    if (obj instanceof User) {
        User other = (User) obj;
        return this.name.equals(other.name);
    }
    return false;

    // 新写法
    if (obj instanceof User other) {
        return this.name.equals(other.name);
    }
    return false;
}

// 实际场景：处理多态对象
public String describe(Object obj) {
    if (obj instanceof String s) return "字符串: " + s;
    if (obj instanceof Integer i) return "整数: " + i;
    if (obj instanceof List<?> list) return "列表: " + list.size() + " 项";
    return "其他: " + obj;
}
```

---

## 关联知识点
