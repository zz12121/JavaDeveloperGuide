---
title: QA_方法引用
tags:
  - Java/Lambda
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-25
---

# 面试题：方法引用

## Q：方法引用有哪几种类型？如何区分？

**A：** 四种类型，核心区分在于"方法属于谁"：

| 类型 | 语法 | 等价 Lambda | 典型场景 |
|------|------|------------|---------|
| 静态方法引用 | `类名::静态方法` | `args → 类名.静态方法(args)` | `Integer::parseInt` |
| 特定对象实例方法 | `实例::实例方法` | `args → 实例.实例方法(args)` | `System.out::println` |
| 任意对象实例方法 | `类名::实例方法` | `(a, rest) → a.实例方法(rest)` | `String::toUpperCase` |
| 构造器引用 | `类名::new` | `args → new 类名(args)` | `ArrayList::new` |

**记忆口诀**：谁的方法，谁就在 `::` 前面。

---

## Q：`String::toUpperCase` 为什么不传入 `this`？

**A：** 这就是**任意对象实例方法引用**的特殊之处。

```java
Function<String, String> f = String::toUpperCase;
f.apply("hello");  // "HELLO"
```

等价于：

```java
Function<String, String> f = s -> s.toUpperCase();
```

**原理**：当方法引用写作 `类名::实例方法` 时，Java 推断第一个参数是方法调用者（即 `this`），不需要显式传入。`toUpperCase` 不需要额外参数，所以 `Function<String, String>` 类型正确。

对比三种调用方式：

```java
// 静态方法：所有参数都是显式传入
Function<String, Integer> f1 = Integer::parseInt;      // 1参数：字符串
BiFunction<String, Integer, String> f2 = String::format; // 2参数：格式和参数

// 任意对象实例方法：第一个参数是调用者（隐式 this）
Function<String, String> f3 = String::toUpperCase;     // 1参数：作为 this
BiFunction<String, String, String> f4 = String::concat; // 2参数：this + 参数

// 特定对象实例方法：所有参数都是显式传入（方法绑定到固定对象）
Consumer<String> c = System.out::println;                // 1参数：println 参数
```

---

## Q：方法引用和 Lambda 哪个性能更好？

**A：** 在功能上完全等价。性能差异来自 **invokedynamic** 指令的 JIT 优化：

```java
list.stream().map(String::trim);  // 编译后：invokedynamic
list.stream().map(s -> s.trim()); // 编译后：invokedynamic
```

**两者都会被 JIT 优化为直接方法调用**，性能几乎没有差别。方法引用的优势在于：
1. **代码更简洁**，减少视觉噪音
2. **编译器优化更容易**，减少推断步骤
3. **更易读**，一眼看出调用的是哪个已有方法

---

## Q：如何用方法引用实现多级排序？

**A：** 结合 `Comparator.comparing` 和方法引用：

```java
List<Person> people = Arrays.asList(
    new Person("Alice", 25),
    new Person("Bob", 20),
    new Person("Alice", 30)
);

// 按姓名升序
people.sort(Comparator.comparing(Person::getName));

// 按姓名升序，姓名相同按年龄升序
people.sort(Comparator.comparing(Person::getName)
                       .thenComparing(Person::getAge));

// 按姓名升序，姓名相同按年龄降序
people.sort(Comparator.comparing(Person::getName)
                       .thenComparing(Person::getAge, Comparator.reverseOrder()));

// 逆序
people.sort(Comparator.comparing(Person::getAge).reversed());
```

**方法引用 `Person::getName`** 属于任意对象实例方法引用，第一个参数 `Person` 是 `comparing` 的输入 `T`。

---

## Q：为什么 `System.out::println` 可以匹配 `Consumer<T>`？

**A：** 因为 `println` 的签名恰好与 `Consumer.accept` 兼容：

```java
// Consumer 的抽象方法
void accept(T t);

// System.out.println 有多个重载：
void println() {}
void println(boolean x) {}
void println(int x)    {}  // ← 匹配
void println(String x) {}  // ← 匹配（自动装箱）

// 当传入 String 时，编译器选择 println(String)
Consumer<String> c = System.out::println;
c.accept("hello");  // 实际调用 System.out.println("hello")
```

**自动装箱的作用**：如果传入的是 `Object`，会匹配 `println(Object)`。编译器通过类型推断选择最具体的重载。

---

## Q：方法引用能用于私有方法吗？

**A：** 可以。方法引用不要求方法是 `public` 的：

```java
public class MyService {
    private String process(String input) {
        return input.toUpperCase();
    }

    public List<String> processAll(List<String> inputs) {
        // 私有方法也可以方法引用
        return inputs.stream()
                     .map(this::process)   // 引用私有方法
                     .collect(Collectors.toList());
    }
}
```

`this::process` 属于特定对象实例方法引用，只要在同一个类或能访问的范围内，私有方法同样适用。
