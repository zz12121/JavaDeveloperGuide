---
title: JDK 8新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口默认方法与静态方法

## Q1：JDK 8 为什么要在接口中引入 default 方法？

**A**：核心原因是**向后兼容**。JDK 8 要在 `Collection` 接口中新增 `stream()` 方法，如果接口只能有抽象方法，所有实现类（`ArrayList`、`HashSet` 等）都必须重写，导致大量已有代码编译失败。default 方法提供默认实现，已有实现类无需任何修改即可编译通过。

---

## Q2：接口的 default 方法和静态方法有什么区别？

**A**：

| 对比项 | default 方法 | static 方法 |
|--------|-------------|-------------|
| 调用方式 | 实例调用：`obj.method()` | 接口名调用：`Interface.method()` |
| 能否被继承 | 能，实现类可直接使用 | 不能，实现类无法调用 |
| 能否被重写 | 能 | 不能（静态方法不参与多态） |

---

## Q3：一个类实现两个接口，两个接口有同名 default 方法怎么办？

**A**：编译器会报错，要求显式解决冲突。两种方式：
1. 重写该方法，自行实现逻辑
2. 在重写方法中通过 `接口名.super.方法名()` 指定调用哪个接口的默认实现
```java
class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // 明确调用 A 的默认实现
    }
}
```

---

## Q4：接口中的 default 方法能否访问实例字段？

**A**：不能。接口不能有实例字段（只能有 `static final` 常量）。default 方法没有 `this` 指向实例状态，它本质上是一个"带默认实现的抽象方法"。

---

## Q5：default 方法和抽象类的区别是什么？什么时候用哪个？

**A**：
- **default 方法**：用于接口功能扩展，不涉及状态。一个类可以实现多个接口。
- **抽象类**：涉及共享状态和通用逻辑时使用，可以有实例字段和构造方法，但只能单继承。
简单原则：**如果只是提供行为，用 default 方法；如果需要状态，用抽象类。**

---

# Lambda 表达式

## Q6：Lambda 表达式的底层原理是什么？它和匿名内部类有什么区别？

**A**：

**底层原理**：Lambda 表达式不是通过匿名内部类实现的，而是通过 `invokedynamic` 字节码指令，在**运行期**由 JDK 的 `LambdaMetafactory.metafactory()` 动态生成实现类。

```java
// 源码
Runnable r = () -> System.out.println("hello");

// 编译后的字节码关键部分
INVOKEDYNAMIC "run()Ljava/lang/Runnable"
    // BootstrapMethods: LAMBDA 运行时生成 lambda$1 implements Runnable
```

**执行流程**：

```
首次执行：JVM → LambdaMetafactory.metafactory() → 动态生成实现类
后续执行：直接使用缓存的实现类
```

**与匿名内部类的核心区别**：

| 对比项 | Lambda | 匿名内部类 |
|--------|--------|-----------|
| 生成时机 | 运行时（invokedynamic） | 编译时（生成 .class 文件）|
| `this` 指向 | 外部类实例 | 匿名内部类自身 |
| 变量捕获 | 只能读 final / effectively final | JDK 8 前可修改局部变量 |
| 编译产物 | 不产生独立 .class 文件 | 产生 `Outer$1.class` 等文件 |
| 性能 | 首次稍慢，后续无额外开销 | 每次 new 一个类 |

> **面试亮点**：Lambda 的 invokedynamic 实现是 Java 语言的重大创新——它推迟了代码生成的时机到运行期，使得 Lambda 不需要任何特殊语法支持，只需要一个函数式接口就能工作。

---

## Q7：Lambda 可以修改外部变量吗？effectively final 是什么？

**A**：

**不能修改**。Lambda 可以读取外部变量，但这些变量必须是 `final` 或 `effectively final`（初始化后不再修改）：

```java
// ✅ effectively final：Lambda 可以读取
int base = 10;
Function<Integer, Integer> add = x -> x + base;  // OK

// ❌ 编译错误：Lambda 不能修改捕获的变量
int count = 0;
Runnable r = () -> count++;  // 编译错误！

// ✅ 正确做法：外部自行处理
int count = 0;
Runnable r = () -> System.out.println(count);
count = 100;  // 外部可以修改，Lambda 读到的值取决于执行时机
```

**effectively final**（JDK 8 引入）：编译器将"没有修改过的局部变量"视为 final，无需显式声明。这是为了让 Lambda 捕获变量时更自然。

---

## Q8：方法引用有哪几种？分别怎么写？

**A**：四类方法引用：

```java
// ① 静态方法引用：ClassName::staticMethod
Function<String, Integer> f1 = Integer::parseInt;

// ② 特定对象的实例方法引用：instance::instanceMethod
String str = "hello";
Callable<Integer> f2 = str::length;

// ③ 任意对象的实例方法引用：ClassName::instanceMethod
Function<String, String> f3 = String::toUpperCase;
// 等价于：s -> s.toUpperCase()

// ④ 构造器引用：ClassName::new
Function<String, Dog> f4 = Dog::new;
```

---

# Stream API

## Q6：Stream 和 Iterator 有什么区别？Stream 有什么优势？

**A**：

| 对比项 | Iterator | Stream |
|--------|---------|--------|
| 数据来源 | 遍历已有集合 | 不存储数据，是视图 |
| 修改源 | 可以修改集合 | 不修改底层数据源 |
| 遍历次数 | 可重复遍历 | 只能消费一次（一次性）|
| 链式操作 | 需要多次循环 | 声明式链式（过滤→映射→聚合）|
| 并行处理 | 需要手动写同步代码 | 一键 `parallelStream()` 并行 |
| 惰性求值 | 即时执行 | 中间操作惰性，终端操作触发 |

**Stream 的核心优势**：将数据处理操作（过滤、映射、排序、分组）以声明式方式链式表达，比嵌套循环 + 临时变量更易读、更易维护。

---

## Q7：Stream 的惰性求值是什么？为什么重要？

**A**：

Stream 的中间操作（如 `filter`、`map`、`distinct`）不会立即执行，只会记录操作。只有遇到**终端操作**（如 `collect`、`forEach`、`count`）时，才会触发整个链的执行：

```java
// 这段代码不会打印任何内容！只是构建了一个 Stream 管道
list.stream()
    .filter(s -> {
        System.out.println("filter: " + s);  // 不会执行
        return s.length() > 3;
    })
    .map(s -> {
        System.out.println("map: " + s);     // 不会执行
        return s.toUpperCase();
    });

// 加上终端操作后才真正执行
list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());  // ← 从这里开始执行
```

**为什么重要**：惰性求值允许 Stream 在遇到终端操作时**一次性优化整个管道**，比如在 filter 后直接 map 可以减少中间对象的创建，提升性能。

---

## Q8：并行流一定比串行流快吗？什么场景下不适用？

**A**：**不一定**。并行流在小数据量、有状态操作等场景下反而可能更慢。

**不适用并行流的场景**：

| 场景 | 原因 |
|------|------|
| 小数据量（< 1000）| 线程创建/管理开销 > 收益 |
| 有状态中间操作（`sorted`、`distinct`）| 需要全量数据收集，并行优势消失 |
| `ArrayList` 以外的数据源 | `ArrayList` 的 `Spliterator` 性能最优 |
| 非线程安全的收集结果 | `ArrayList` 等非线程安全集合在并行写入时需额外同步 |
| CPU 密集型（短任务）| ForkJoinPool 默认并行度 = CPU 核心数，短任务切换开销大 |

```java
// ⚠️ 并行流的常见错误
List<String> unsafe = new ArrayList<>();
list.parallelStream()
    .forEach(unsafe::add);  // 危险！并行写入 ArrayList 不安全

// ✅ 正确：用线程安全的收集器
List<String> safe = list.parallelStream()
    .collect(Collectors.toList());  // 线程安全
```

**何时用并行流**：数据量大（> 10 万）、元素独立（如 ETL 计算、统计聚合）、无状态操作。

---

## Q9：Stream 中 map 和 flatMap 有什么区别？

**A**：

- `map`：一对一转换，每个输入元素产生一个输出元素
- `flatMap`：一对多转换，每个输入元素产生**零到多个**输出元素，合并为一个流

```java
List<String> words = List.of("hello", "world");

// map：每个单词转成字符列表（嵌套）
List<String[]> chars = words.stream()
    .map(w -> w.split(""))   // Stream<String[]>
// [ ["h","e","l","l","o"], ["w","o","r","l","d"] ]

// flatMap：每个单词展开为字符流，再合并为一个流
List<String> flatChars = words.stream()
    .flatMap(w -> Stream.of(w.split("")))  // Stream<String>
// ["h","e","l","l","o","w","o","r","l","d"]
```

> **记忆口诀**：`map` 是一对一，`flatMap` 是把嵌套的流"压平"成一层的流。

**A**：
- **default 方法**：用于接口功能扩展，不涉及状态。一个类可以实现多个接口。
- **抽象类**：涉及共享状态和通用逻辑时使用，可以有实例字段和构造方法，但只能单继承。
简单原则：**如果只是提供行为，用 default 方法；如果需要状态，用抽象类。**
