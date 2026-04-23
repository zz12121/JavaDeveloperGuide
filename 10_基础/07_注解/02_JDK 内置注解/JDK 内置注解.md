---
title: JDK 内置注解
tags:
  - Java/注解
  - 原理型
module: 07_注解
created: 2026-04-18
---

# JDK 内置注解

## 核心结论

JDK 内置了 7 个注解，其中 3 个位于 `java.lang` 包，4 个位于 `java.lang.annotation` 包（元注解）。最常用的是 `@Override`、`@Deprecated`、`@SuppressWarnings`。

## 深度解析

### java.lang 包中的 3 个内置注解

#### @Override
- **作用**：标记方法覆盖了父类（或接口）的方法
- **编译器检查**：如果标记的方法并非真正重写了父类方法，编译报错
- **保留策略**：`SOURCE`（仅编译期检查）

```java
@Override
public String toString() {
    return "User{name=" + name + "}";
}

// 以下会编译报错：方法未覆盖父类方法
@Override
public void printInfo() { }
```

#### @Deprecated
- **作用**：标记已过时的元素（类、方法、字段），不建议使用
- **保留策略**：`RUNTIME`
- **效果**：编译时产生警告，IDE 中显示删除线

```java
@Deprecated
public void oldMethod() {
    // 已废弃，请使用 newMethod()
}
```

#### @SuppressWarnings
- **作用**：抑制编译器警告
- **保留策略**：`SOURCE`
- **常用参数**：

| 参数值 | 抑制的警告类型 |
|--------|---------------|
| `unchecked` | 未检查的类型转换（泛型擦除相关） |
| `deprecation` | 使用了已过时的 API |
| `rawtypes` | 原始类型（未指定泛型参数） |
| `unused` | 未使用的变量/方法 |
| "all" | 所有警告 |

```java
@SuppressWarnings("unchecked")
List list = new ArrayList(); // 抑制泛型警告
```

### java.lang.annotation 包中的 4 个元注解



## 关联知识点