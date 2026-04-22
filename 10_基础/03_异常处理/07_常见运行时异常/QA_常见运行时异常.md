---
title: 常见运行时异常
tags:
  - Java/异常处理
  - 原理型
  - 问答
module: 03_异常处理
created: 2026-04-18
---

# 常见运行时异常

## Q1：列举你知道的常见 RuntimeException

**A：**

| 异常 | 场景 |
| ------ | ------ |
| `NullPointerException` | null 引用调用方法 |
| `ArrayIndexOutOfBoundsException` | 数组下标越界 |
| `ClassCastException` | 类型强转失败 |
| `NumberFormatException` | 字符串转数字格式非法 |
| `ArithmeticException` | 整数除零 |
| `IllegalArgumentException` | 参数非法 |
| `ConcurrentModificationException` | 遍历集合时修改 |
| `UnsupportedOperationException` | 不可变集合操作 |
| `StackOverflowError` | 无限递归（注意：是 Error） |

---

## Q2：for-each 遍历 List 时删除元素为什么会抛异常？如何解决？

**A：**
for-each 底层是 `Iterator` 实现，每次修改集合会改变 `modCount`，而 Iterator 内部记录了期望的 `expectedModCount`，两者不一致时抛 `ConcurrentModificationException`（fail-fast 机制）。

**解决方案：**
```java
// 方案1：Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) it.remove();
}

// 方案2：removeIf（JDK 8+，推荐）
list.removeIf(s -> "b".equals(s));

// 方案3：倒序遍历（for 循环）
for (int i = list.size() - 1; i >= 0; i--) {
    if ("b".equals(list.get(i))) list.remove(i);
}

// 方案4：CopyOnWriteArrayList（线程安全场景）
```

---

## Q3：NullPointerException 如何避免？

**A：**
1. **提前判空**：`if (obj != null) { ... }`
2. **Optional**：`Optional.ofNullable(obj).orElse("default")`
3. **Objects.requireNonNull**：快速失败，在方法入口校验参数
4. **@NonNull 注解**：配合 Lombok/IDE 静态检查
5. **JDK 14+ Helpful NPE**：精确定位是哪个变量为 null

---

## Q4：StackOverflowError 和 OutOfMemoryError 有什么区别？

**A：**

| | `StackOverflowError` | `OutOfMemoryError` |
|--|----------------------|-------------------|
| 触发原因 | 方法调用栈太深（无限递归） | 堆内存不足 |
| 内存区域 | JVM 栈 | 堆 |
| 是否可恢复 | 一般不可恢复 | 一般不可恢复 |
| 常见场景 | 无终止条件的递归 | 对象创建过多/内存泄漏 |

---

## Q5：`1/0` 会抛什么异常？`1.0/0` 呢？

**A：**
- `1/0`：整数除零，抛 `ArithmeticException: / by zero`
- `1.0/0`：浮点数除零，**不抛异常**，返回 `Infinity`（无穷大）
- `0.0/0`：返回 `NaN`（Not a Number）

这是一道经典细节题，浮点数运算遵循 IEEE 754 标准，不抛异常。
