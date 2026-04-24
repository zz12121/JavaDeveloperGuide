---
title: Lambda 表达式作用域
tags:
  - Java/Lambda
  - 原理型
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# Lambda 表达式作用域

## Q：Lambda 可以访问外部变量吗？有什么限制？

**A：** Lambda 可以访问：
- ✅ **外部类的成员变量**（可以读写，无限制）
- ✅ **方法的局部变量**（可以读，但必须是 **effectively final**）
- ❌ **不能修改**在 Lambda 中引用的局部变量

```java
int factor = 2;           // effectively final
Function<Integer, Integer> f = x -> x * factor; // ✅
// factor = 3;            // 编译报错：被 Lambda 引用后不能修改
```

## Q：为什么 Lambda 中访问的局部变量必须是 effectively final？

**A：** 三个原因：
1. **值捕获**：Lambda 捕获的是局部变量的**值拷贝**，不是引用。如果允许修改原变量，拷贝值与原变量不一致
2. **线程安全**：Lambda 可能在另一个线程执行，允许修改局部变量需要额外同步
3. **简化模型**：避免并发场景下的可见性问题

成员变量不存在这个问题，因为成员变量存在于堆中，Lambda 和外部方法共享同一个对象引用。

## Q：Lambda 中的 this 指向什么？

**A：** Lambda 中的 `this` 指向**包含 Lambda 的外部类对象**，不是 Lambda 自身（Lambda 不是匿名内部类，没有自己的 this）。这是 Lambda 和匿名内部类的一个重要区别。

```java
public class Outer {
    public void test() {
        Runnable r = () -> System.out.println(this); // 指向 Outer 实例
    }
}
```

## Q：Lambda 中需要修改外部变量怎么办？

**A：** 使用可变对象包装：
```java
AtomicInteger count = new AtomicInteger(0);
list.forEach(x -> count.incrementAndGet());  // 线程安全
```
