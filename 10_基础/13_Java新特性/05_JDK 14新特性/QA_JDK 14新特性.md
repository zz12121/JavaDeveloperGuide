---
title: JDK 14新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Switch 表达式

## Q1：Switch 表达式相比旧版 switch 有哪些改进？

**A**：
1. **箭头语法**：`case x ->` 替代 `case x:`，无需 `break`，消除 fall-through
2. **返回值**：switch 本身可以作为表达式，直接返回值
3. **多常量匹配**：`case MONDAY, FRIDAY ->` 一行匹配多个值
4. **独立作用域**：箭头语法每个 case 有独立变量作用域
5. **穷举检查**：编译器会检查是否覆盖所有情况（枚举类型）

---

## Q2：yield 和 return 有什么区别？

**A**：
- `yield`：从 **switch 表达式**的代码块中返回值
- `return`：从**方法**中返回值
```java
String result = switch (day) {
    case MONDAY -> {
        // return "忙碌";  // ❌ 编译错误：试图从方法返回
        yield "忙碌";      // ✅ 从 switch 表达式返回
    }
    default -> "普通";
};
```

---

## Q3：新语法中还需要 break 吗？

**A**：箭头语法（`->`）不需要 break。但如果使用传统冒号语法（`:`），仍然需要 break 来防止穿透：
```java
// 箭头语法：不需要 break
case MONDAY -> "忙碌";

// 冒号语法（仍支持）：需要 break
case MONDAY: yield "忙碌";  // yield 替代了 break + 赋值
```

---

## Q4：switch 表达式可以处理 null 吗？

**A**：不能。对 null 调用 switch 会抛 `NullPointerException`。如果需要处理 null，可以：
```java
String result = switch (obj != null ? obj : "NULL_CASE") {
    case "NULL_CASE" -> "空值";
    case String s -> s;
    default -> obj.toString();
};
```


# Pattern Matching for instanceof

## Q1：Pattern Matching for instanceof 是什么？

**A**：JDK 16 正式引入的特性，允许在 `instanceof` 检查类型的同时声明变量并自动转型，一行代码完成"类型判断 + 变量声明 + 强制转型"三个操作。
```java
// 旧写法
if (obj instanceof String) {
    String s = (String) obj;
}

// 新写法
if (obj instanceof String s) {
    // s 已经是 String 类型，可直接使用
}
```

---

## Q2：模式变量的作用域是什么？

**A**：模式变量（如 `String s`）仅在条件为 true 的分支内有效。且在 `&&` 右侧也可使用（因为短路求值保证安全），但在 `||` 右侧不可使用。
```java
if (obj instanceof String s && s.length() > 5) { ... }  // ✅
if (obj instanceof String s || s.length() > 5) { ... }  // ❌
```

---

## Q3：模式变量可以被重新赋值吗？

**A**：不能。模式变量隐式为 `final`，编译器不允许重新赋值。
```java
if (obj instanceof String s) {
    s = "new";  // ❌ 编译错误
}
```

---

## Q4：这个特性和 Switch 表达式有什么关系？

**A**：Pattern Matching for instanceof 先在 `if` 语句中引入模式匹配，后来这个能力扩展到了 `switch` 表达式（JDK 17+ 预览），允许在 switch 的 case 中使用类型模式：
```java
return switch (obj) {
    case String s -> "字符串: " + s;
    case Integer i -> "整数: " + i;
    default -> obj.toString();
};
```

---

## Q5：从哪个 JDK 版本开始可用？

**A**：JDK 14 预览，JDK 15 二次预览，**JDK 16 正式发布**。