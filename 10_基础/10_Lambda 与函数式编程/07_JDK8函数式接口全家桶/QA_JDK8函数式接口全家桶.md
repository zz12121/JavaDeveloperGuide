---
title: QA_JDK8函数式接口全家桶
tags:
  - Java/Lambda
  - 问答
module: 10_Lambda与函数式编程
created: 2026-04-25
---

# 面试题：JDK8函数式接口全家桶

## Q：`Function.identity()` 和 `t -> t` 有什么区别？

**A：** 功能完全等价，但 `identity()` 有优化：

```java
// 两者效果一样
Function<String, String> f1 = Function.identity();
Function<String, String> f2 = t -> t;

// identity() 的优势：
// 1. 代码意图更清晰
// 2. JDK 内部对 identity() 有缓存，不会重复创建
Stream<String> s1 = list.stream().map(Function.identity());

// 3. 在 key 映射等场景中更标准
Map<String, List<Person>> byName = people.stream()
    .collect(Collectors.groupingBy(Person::getName));
// 注意：这里 groupingBy 的第二个参数是 Function，不是 UnaryOperator

// 用作 BinaryOperator 的初始值时更规范
UnaryOperator<String> identity = Function.identity();
```

---

## Q：为什么需要 `ToIntFunction` 而不是直接用 `Function<T, Integer>`？

**A：** 避免 **自动装箱/拆箱** 的性能开销：

```java
// 普通 Function：涉及装箱
Function<String, Integer> f1 = String::length;
// 调用时：int → Integer（装箱），返回时：Integer → int（拆箱）

// ToIntFunction：无装箱
ToIntFunction<String> f2 = String::length;
f2.applyAsInt("hello");  // 直接返回 int，无装箱

// Stream 中的性能差异
list.stream()
    .mapToInt(String::length)        // ToIntFunction，等价但更快
    .sum();

// JDK 推荐：
// - 涉及数学计算（加减乘除）→ 用基本类型特化（ToIntFunction 等）
// - 只是简单传递 → 用普通 Function
```

**Integer vs int 的装箱开销**：每百万次操作约增加 1-2ms，在高性能场景下（大数据流处理）差异显著。

---

## Q：`Consumer.andThen()` 如果第一个 Consumer 抛异常会怎样？

**A：** 第二个 Consumer **不会执行**，异常会向外传播：

```java
Consumer<String> first = s -> {
    System.out.println("first: " + s);
    throw new RuntimeException("first failed");
};
Consumer<String> then = s -> System.out.println("then: " + s);

try {
    first.andThen(then).accept("hello");
} catch (RuntimeException e) {
    System.out.println("then was NOT executed"); // 确实未执行
}
```

**解决方案：使用 `andThen` 时确保每个 Consumer 独立，或者在内部捕获异常**：

```java
// 方案1：每个 Consumer 独立处理
Consumer<String> safe = s -> {
    try {
        first.accept(s);
    } catch (Exception ignored) {}
    then.accept(s);
};

// 方案2：使用 try-with-resources（如果实现了 Closeable）
```

---

## Q：`Predicate.not()` 在 JDK 8 和 JDK 11+ 有什么区别？

**A：** JDK 8 没有 `Predicate.not()`，JDK 11+ 新增：

```java
// JDK 8：需要用 negate()
Predicate<String> notBlank = s -> !s.isBlank();
Predicate<String> blank = notBlank.negate();

// JDK 11+：更简洁
Predicate<String> blank = Predicate.not(String::isBlank);

// Stream 中的实际应用
list.stream()
    .filter(Predicate.not(String::isBlank))   // JDK 11+，更可读 ✅
    .filter(s -> !s.isBlank())                  // JDK 8+
    .collect(Collectors.toList());
```

**`Predicate.not()` 的优势**：当传入方法引用时，`not()` 比 `negate()` 更简洁，尤其在方法引用场景下可读性更好。

---

## Q：`BinaryOperator.maxBy()` 和 `Comparator.comparing()` 有什么关系？

**A：** `BinaryOperator.maxBy()` 是 `Comparator.comparing()` 的快捷方式：

```java
// 方式1：BinaryOperator + Comparator
Comparator<String> byLen = Comparator.comparingInt(String::length);
BinaryOperator<String> longest = BinaryOperator.maxBy(byLen);
longest.apply("abc", "ab");  // "abc"

// 方式2：直接用 BinaryOperator 的静态方法
BinaryOperator<String> longer = (a, b) ->
    a.length() > b.length() ? a : b;  // 手写逻辑
longer.apply("abc", "ab");  // "abc"

// Stream.reduce 中的使用
list.stream()
    .reduce(BinaryOperator.maxBy(Comparator.comparingInt(String::length)));
```

**常见笔试题**：
```java
// 求最大值（reduce 的正确写法）
String longest = list.stream()
    .reduce(BinaryOperator.maxBy(
        Comparator.comparingInt(String::length)))
    .orElse("");

// 错误写法：reduce 直接用 maxBy 是函数式风格，但不是 reduce 的语义
String wrong = list.stream().max(String::compareTo).orElse("");
```

---

## Q：`IntFunction<R>` 和 `Function<Integer, R>` 什么时候用？

**A：** 优先使用基本类型特化（`IntFunction` 等），避免装箱：

```java
// IntFunction<R>：接收 int，返回 R，无装箱
IntFunction<String[]> arrayCreator = size -> new String[size];
String[] arr = arrayCreator.apply(10);  // 创建长度为10的数组

// Function<Integer, R>：接收 Integer，有装箱
Function<Integer, String[]> arrayCreator2 = size -> new String[size];
// size 自动装箱为 Integer，apply 返回时 Integer 拆箱

// 典型场景：Collectors.toMap 的 keyMapper/valueMapper
Map<String, Integer> map = list.stream()
    .collect(Collectors.toMap(
        Function.identity(),   // key：String → String
        String::length         // value：String → int（自动装箱为 Integer）
    ));

// 高性能场景
IntFunction<List<String>> listCreator = ArrayList::new;
List<String>[] arrays = listCreator.apply(5);  // 创建 5 个 ArrayList
```

---

## Q：`UnaryOperator` 和 `Function<T, T>` 完全等价吗？

**A：** 功能等价，但类型语义不同：

```java
// 完全等价，UnaryOperator 是 Function 的子接口
UnaryOperator<String> u1 = String::toUpperCase;
Function<String, String> f1 = String::toUpperCase;

// 类型不能互换赋值（虽然可以相互调用）
UnaryOperator<String> u2 = s -> s.trim();
// Function<String, String> f2 = u2; // 编译错误！类型不兼容

// UnaryOperator 的优势：
// 1. 代码意图更清晰（入参和返回类型相同，操作型语义）
// 2. Stream.reduce 默认使用 BinaryOperator
Stream<String> s1 = Stream.of("a", "b", "c");
String reduced = s1.reduce("", (a, b) -> a + b);  // BinaryOperator

// Map 替换值（UnaryOperator 更自然）
Map<String, Integer> counts = new HashMap<>();
counts.computeIfPresent("key", (k, v) -> v + 1);  // BiFunction
// 注意：computeIfPresent 的第三个参数是 BiFunction，不是 UnaryOperator
```
