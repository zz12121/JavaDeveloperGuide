---
title: Lambda 表达式作用域
tags:
  - Java/Lambda
  - 原理型
module: 10_Lambda与函数式编程
created: 2026-04-18
---

# Lambda 表达式作用域

## 核心结论

Lambda 表达式可以访问外部类的成员变量和方法的局部变量，但访问局部变量时该变量必须是**effectively final**（事实上的 final）的。Lambda 中的 `this` 指向包含它的外部类对象，而非匿名内部类对象。

## 深度解析

### 访问外部变量

```java
public class LambdaScope {
    private String name = "外部成员变量";

    public void test() {
        String local = "局部变量";        // effectively final
        final String constant = "常量";

        Consumer<String> c = s -> {
            // ✅ 访问成员变量
            System.out.println(name);
            // ✅ 修改成员变量
            name = "修改后的成员变量";
            // ✅ 访问局部变量
            System.out.println(local);
            // ✅ 访问常量
            System.out.println(constant);
            // ❌ 修改局部变量 → 编译报错
            // local = "new"; // Variable used in lambda expression should be final or effectively final
        };
    }
}
```

### effectively final 规则

局部变量在 Lambda 中被引用后，不能再被修改（即使没有 final 修饰）：
```java
int num = 10;         // effectively final（后续没修改）
Consumer<Integer> c = x -> System.out.println(x + num);
// num = 20;          // 编译报错！num 在 Lambda 中被引用，不能再修改
```

```java
int num = 10;
num = 20;            // 在 Lambda 之前修改了
Consumer<Integer> c = x -> System.out.println(x + num); // ✅ 可以，但 num=20 被固定
// num = 30;          // 编译报错
```

### 为什么要求 effectively final

1. **线程安全**：Lambda 可能在另一个线程执行，如果允许修改局部变量，需要额外的同步机制
2. **值传递**：Lambda 捕获的是变量的**值**（拷贝），不是引用。如果允许修改，拷贝的值和原变量不一致，造成混乱
3. **简化模型**：避免并发场景下的变量可见性问题

### this 关键字的指向

```java
public class Outer {
    private String name = "Outer";

    public void test() {
        // 匿名内部类中的 this 指向匿名类自身
        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                System.out.println(this.getClass().getName()); // 匿名类名
            }
        };

        // Lambda 中的 this 指向外部类
        Runnable r2 = () -> {
            System.out.println(this.getClass().getName()); // Outer
            System.out.println(this.name);                  // Outer
        };
    }
}
```

### 解决需要修改外部变量的方案

```java
// 方案 1：使用数组（不推荐）
int[] counter = {0};
Consumer<Integer> c = x -> counter[0] += x;

// 方案 2：使用 Atomic 类（线程安全）
AtomicInteger counter = new AtomicInteger(0);
Consumer<Integer> c = x -> counter.addAndGet(x);

// 方案 3：将变量提升为成员变量
```

## 关联知识点

