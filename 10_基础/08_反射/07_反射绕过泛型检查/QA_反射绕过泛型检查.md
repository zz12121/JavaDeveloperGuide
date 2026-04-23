---
title: 反射绕过泛型检查
tags:
  - Java/反射
  - 场景型
  - 问答
module: 08_反射
created: 2026-04-18
---

# 反射绕过泛型检查

## Q：反射为什么能绕过泛型检查？

**A：** 因为 Java 泛型在编译期发生**类型擦除**，运行时泛型信息不存在。`List<String>` 和 `List<Integer>` 在运行时都是 `ArrayList`。反射调用的是原始类型的 `add(Object)` 方法，该方法没有泛型约束，所以可以添加任意类型。
```java
List<String> list = new ArrayList<>();
Method add = list.getClass().getMethod("add", Object.class);
add.invoke(list, 123);  // 成功添加 Integer
```

## Q：绕过泛型后会有什么问题？

**A：** 编译时不会报错，但**运行时**在取出元素并转为目标类型时会抛出 `ClassCastException`：
```java
List<String> list = new ArrayList<>();
// 反射添加 Integer
list.getClass().getMethod("add", Object.class).invoke(list, 123);

// 遍历时 String s = (String) iterator.next() → ClassCastException
for (String s : list) { } // 抛出 ClassCastException
```

## Q：有没有办法在运行时获取泛型的实际类型参数？

**A：** 对于**声明在类/方法签名上的泛型**可以通过反射获取，但对于**使用时赋值给变量的泛型类型**无法获取：
```java
// 可以获取：类声明上的泛型
public class GenericClass<T> {}
ParameterizedType type = (ParameterizedType) GenericClass.class.getGenericSuperclass();
Type actualType = type.getActualTypeArguments()[0]; // 获取 T 的实际类型

// 无法获取：局部变量上的泛型
List<String> list = new ArrayList<>(); // 运行时无法知道 <String>
```
