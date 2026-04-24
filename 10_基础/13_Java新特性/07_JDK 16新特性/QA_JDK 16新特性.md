---
title: JDK 16新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Records 记录类

## Q1：Record 是什么？解决了什么问题？

**A**：Record 是 JDK 16 正式引入的语法，用于快速定义**不可变数据载体类**。编译器自动生成构造方法、访问器方法、equals、hashCode、toString，将原来几十行的 POJO 压缩为一行声明。
```java
// 一行定义
public record Point(int x, int y) {}

// 自动拥有：构造方法、x()、y()、equals()、hashCode()、toString()
```

---

## Q2：Record 可以继承其他类吗？可以实现接口吗？

**A**：
- **不能继承其他类**：Record 隐式 `final`，且隐式继承 `java.lang.Record`
- **可以实现接口**：完全支持

```java
public record User(String name) implements Serializable {}
```

---

## Q3：Record 的访问器方法为什么是 x() 而不是 getX()？

**A**：这是 Record 的设计选择。Record 定位于**不可变数据载体**，不是 JavaBean。JavaBean 规范的 `getX()` 风格暗示可变性（有对应的 `setX()`），而 Record 没有 setter。使用 `x()` 风格更简洁，也更接近函数式编程的命名习惯。

---

## Q4：Record 可以自定义方法吗？

**A**：可以。可以在 Record 中声明静态方法、实例方法，也可以重写自动生成的方法：

```java
public record Range(int min, int max) {
    // 紧凑构造方法（验证逻辑）
    public Range {
        if (min > max) throw new IllegalArgumentException();
    }

    // 自定义实例方法
    public int length() { return max - min; }

    // 静态工厂方法
    public static Range zero() { return new Range(0, 0); }
}
```

---

## Q5：Record 和 Lombok @Data 怎么选？

**A**：
- **Record**：JDK 16+、数据不可变、无第三方依赖 → 选 Record
- **Lombok @Data**：需要可变性、JDK 版本低、已有大量 Lombok 代码 → 继续用 Lombok
Record 是不可变的，如果需要 setter 或可变字段，不适合用 Record。

---

## Q6：Record 从哪个 JDK 版本开始可用？

**A**：JDK 14 预览，JDK 15 二次预览，**JDK 16 正式发布**。生产环境至少需要 JDK 16。
