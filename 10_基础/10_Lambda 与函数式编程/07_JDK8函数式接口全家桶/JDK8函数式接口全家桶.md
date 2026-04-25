---
title: JDK8函数式接口全家桶
tags:
  - Java/Lambda
  - 原理型
module: 10_Lambda与函数式编程
created: 2026-04-25
---

# JDK8函数式接口全家桶

## 核心结论

JDK 8 在 `java.util.function` 包中提供了 **43 个**函数式接口，覆盖了从简单到复杂、从一元到多元的各种场景。核心四大接口（Consumer/Supplier/Predicate/Function）衍生出大量变体，掌握其命名规律就能快速找到需要的接口。

## 接口分类总览

```
java.util.function
├── Consumer（消费型）
│   ├── Consumer<T>
│   ├── BiConsumer<T,U>
│   ├── IntConsumer / LongConsumer / DoubleConsumer（基本类型特化）
│   └── ObjIntConsumer<T> / ObjLongConsumer<T> / ObjDoubleConsumer<T>
├── Supplier（供给型）
│   ├── Supplier<T>
│   └── IntSupplier / LongSupplier / DoubleSupplier / BooleanSupplier
├── Predicate（断言型）
│   ├── Predicate<T>
│   ├── BiPredicate<T,U>
│   └── IntPredicate / LongPredicate / DoublePredicate
├── Function（函数型）
│   ├── Function<T,R>
│   ├── BiFunction<T,U,R>
│   ├── IntFunction<R> / LongFunction<R> / DoubleFunction<R>
│   ├── IntToLongFunction / IntToDoubleFunction
│   ├── LongToIntFunction / LongToDoubleFunction
│   ├── DoubleToIntFunction / DoubleToLongFunction
│   └── ToIntFunction<T> / ToLongFunction<T> / ToDoubleFunction<T>
├── Operator（操作型）
│   ├── UnaryOperator<T>  extends Function<T,T>
│   ├── BinaryOperator<T> extends BiFunction<T,T,T>
│   └── IntUnaryOperator / LongUnaryOperator / DoubleUnaryOperator
│   └── IntBinaryOperator / LongBinaryOperator / DoubleBinaryOperator
└── 其他
    ├── Runnable / Callable<V>（已有）
    ├── Comparator<T>
    └── ActionListener / WindowListener（GUI，已少用）
```

## Consumer 系列（消费，无返回）

```java
// 基本型：接收 T，返回 void
Consumer<String> c = s -> System.out.println(s);
c.accept("hello");

// 双参数型：接收 T 和 U
BiConsumer<String, Integer> bc = (name, age) ->
    System.out.println(name + ": " + age);
bc.accept("Alice", 25);

// andThen：先执行当前，再执行 after
Consumer<String> first = s -> System.out.print("[1]" + s);
Consumer<String> then  = s -> System.out.print("[2]" + s);
first.andThen(then).accept("hello");  // [1]hello[2]hello

// andThen 异常处理
// 注意：如果 first 抛异常，then 不会执行
```

## Supplier 系列（供给，无参数）

```java
// 基本型：无参数，返回 T
Supplier<LocalDate> dateSupplier = LocalDate::now;
Supplier<List<String>> listFactory = ArrayList::new;

// 基本类型特化（避免装箱）
DoubleSupplier ds = Math::random;  // () -> double
IntSupplier is = () -> 42;

// BooleanSupplier（用于条件判断）
BooleanSupplier bs = () -> Math.random() > 0.5;
```

## Predicate 系列（断言，返回 boolean）

```java
// 基本型
Predicate<String> isEmpty = s -> s.isEmpty();
Predicate<String> notEmpty = isEmpty.negate();

// 链式组合
Predicate<String> isNotBlank = Predicate.not(String::isBlank);
Predicate<String> startsWithA = s -> s.startsWith("A");
isNotBlank.and(startsWithA).test("Apple");  // true

// or / and / negate
Predicate<String> valid = isNotBlank.or(s -> s.equals("default"));

// isEqual（JDK 11+，推荐替代 equals）
Predicate<String> isBob = Predicate.isEqual("Bob");
Predicate<String> isBob2 = Predicate.isEqual("Bob"); // static 方法

// BiPredicate：两个参数
BiPredicate<String, Integer> nameAndAge = (name, age) ->
    name.length() > 0 && age > 0;
```

## Function 系列（函数，转换）

```java
// 基本型：T → R
Function<String, Integer> strLen = String::length;

// compose vs andThen（方向相反）
Function<Integer, String> intToStr = i -> "num=" + i;
Function<String, String> trimAndWrap = s -> "[" + s.trim() + "]";

// andThen：先当前，再 after
String r1 = strLen.andThen(intToStr).apply("hello");  // "num=5"
// 5 → "num=5"

// compose：先 before，再当前
String r2 = strLen.compose(trimAndWrap).apply("  hi  ");
// "  hi  " → "hi" → 2

// identity
Function<String, String> identity = Function.identity();
// 等价于 s -> s

// ToIntFunction 等（输出是基本类型）
ToIntFunction<String> toInt = String::length;
toInt.applyAsInt("hello");  // 5（返回 int，无装箱）
```

## Operator 系列（操作，继承自 Function）

```java
// UnaryOperator<T> = Function<T, T>
UnaryOperator<String> upper = String::toUpperCase;
UnaryOperator<String> trim = String::trim;

// BinaryOperator<T> = BiFunction<T, T, T>
BinaryOperator<Integer> max = Integer::max;
BinaryOperator<Integer> sum = Integer::sum;
BinaryOperator<String> longer = (a, b) ->
    a.length() > b.length() ? a : b;

// maxBy / minBy（JDK 8+，按比较器取最大/最小）
Comparator<String> byLen = Comparator.comparingInt(String::length);
BinaryOperator<String> longest = BinaryOperator.maxBy(byLen);
longest.apply("ab", "abcde");  // "abcde"

// 基本类型特化
IntUnaryOperator inc = i -> i + 1;
IntBinaryOperator add = Integer::sum;
```

## 常用场景速查表

| 场景 | 接口 | 示例 |
|------|------|------|
| forEach 遍历 | `Consumer<T>` | `list.forEach(System.out::println)` |
| Stream.filter | `Predicate<T>` | `filter(s -> !s.isBlank())` |
| Stream.map | `Function<T,R>` | `map(String::toUpperCase)` |
| Stream.findAny.orElse | `Supplier<T>` | `orElseGet(() -> getDefault())` |
| 比较器组合 | `Comparator<T>` + `Function` | `Comparator.comparing(User::getName)` |
| 条件执行 | `Supplier<Boolean>` | `() -> checkCondition()` |
| 双重条件 | `BiPredicate<T,U>` | `(name, age) -> name.length() > 0 && age > 18` |
| 累加计算 | `BinaryOperator<T>` | `reduce(0, Integer::sum)` |
| 类型转换（避免装箱） | `ToIntFunction<T>` | `mapToInt(ToIntFunction)` |

## 关联知识点

- [[四大核心函数式接口]] — Consumer/Supplier/Predicate/Function 详解
- [[Lambda 表达式作用域]] — Lambda 与函数式接口的变量捕获
