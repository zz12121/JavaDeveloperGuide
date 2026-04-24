---
title: JDK 9新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口私有方法

## Q1：JDK 9 为什么要在接口中引入 private 方法？

**A**：JDK 8 的 default 方法解决了接口功能扩展问题，但当多个 default 方法有重复代码时，需要一个内部复用机制。不能把重复代码提取为新的 default 方法（会暴露给实现类），也不能用 static 方法（无法访问实例上下文）。private 方法正好填补了这个空白。

---

## Q2：接口中的 private 方法和 private static 方法有什么区别？

**A**：

| 对比项 | `private` 方法 | `private static` 方法 |
|--------|---------------|---------------------|
| 归属 | 实例级别 | 类级别 |
| 调用者 | default 方法、其他 private 实例方法 | default 方法、static 方法、private 方法 |
| 能否调用 default 方法 | ❌ 不能 | ❌ 不能 |
| 能否访问实例相关内容 | 可以（虽然接口没有实例字段） | 不能 |

---

## Q3：实现类能访问接口中的 private 方法吗？

**A**：不能。private 方法完全属于接口内部实现细节，实现类无法看到、无法调用、也无法重写。这和类的 private 方法一样遵循封装原则。

---

## Q4：private 方法在接口中的使用场景是什么？

**A**：主要用于多个 default 方法之间的**代码复用**。例如：

```java
interface Logger {
    default void info(String msg) { log("INFO", msg); }
    default void error(String msg) { log("ERROR", msg); }
    default void warn(String msg) { log("WARN", msg); }

    // 三个 default 方法共享同一个 private 实现
    private void log(String level, String msg) {
        System.out.println("[%s] %s: %s".formatted(level, timestamp(), msg));
    }
}
```


---

# 集合工厂方法

## Q1：List.of() 和 Arrays.asList() 有什么区别？

**A**：

| 对比项 | `List.of()` | `Arrays.asList()` |
|--------|-------------|-------------------|
| 可变性 | 完全不可变 | 固定大小，但元素可 set |
| null 元素 | ❌ 不允许 | ✅ 允许 |
| 与原数组关系 | 独立 | 共享底层数组 |
| JDK 版本 | JDK 9 | JDK 1.2 |
```java
var a = Arrays.asList("A", "B");
a.set(0, "X");   // ✅ 可以修改元素

var b = List.of("A", "B");
b.set(0, "X");   // ❌ UnsupportedOperationException
```

---

## Q2：List.of() 创建的集合可以修改吗？

**A**：完全不可修改。不能增（add）、不能删（remove）、不能改（set），任何修改操作都会抛 `UnsupportedOperationException`。如果需要可变集合，可以包装一下：
```java
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
```

---

## Q3：List.of() 为什么不允许 null？

**A**：这是 JDK 的设计决策。不可变集合如果允许 null 元素，会导致 `contains(null)` 等方法的行为需要额外处理（返回 true 还是 false？）。禁止 null 可以简化实现，避免歧义。如需 null，使用 `ArrayList` 或 `new CopyOnWriteArrayList<>()`。

---

## Q4：Map.of() 最多支持多少个键值对？

**A**：`Map.of()` 重载方法最多支持 **10 个键值对**（5 对参数版本）。超过 10 个使用 `Map.ofEntries(Map.entry(...), ...)` 或 `Map.copyOf(map)`。
