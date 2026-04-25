---
title: JDK 8新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# 接口默认方法与静态方法

## 核心结论

JDK 8 允许接口中定义 **default 方法**和 **static 方法**。default 方法有默认实现，实现类可以直接继承使用或重写，解决了"接口升级"的兼容性问题。static 方法属于接口本身，只能通过接口名调用。

---

## 深度解析

### 1. 为什么需要 default 方法？

JDK 8 引入 `Stream` 时，需要在 `Collection` 接口中添加 `stream()` 和 `parallelStream()` 方法。如果接口只能有抽象方法，那么**所有实现类都必须重写**，导致大量已有代码编译失败。

```java
// JDK 8 之前：接口只有抽象方法，添加方法 = 破坏所有实现类
public interface Collection {
    // 新增 stream() → 所有 List/Set/Queue 实现类都要改
}

// JDK 8 解决方案：default 方法提供默认实现
public interface Collection {
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
}
```

### 2. default 方法语法

```java
public interface MyInterface {
    // 抽象方法（传统）
    void abstractMethod();

    // default 方法（JDK 8）
    default void defaultMethod() {
        System.out.println("默认实现");
    }

    // 可以调用其他 default 方法
    default void combinedMethod() {
        defaultMethod();  // ✅ 接口内部调用
    }
}
```

### 3. 接口静态方法

```java
public interface MyInterface {
    // 静态方法（JDK 8）
    static void staticMethod() {
        System.out.println("接口静态方法");
    }
}

// 调用方式
MyInterface.staticMethod();  // ✅ 通过接口名调用
```

> ⚠️ 接口静态方法**不能被实现类继承**，这与类的静态方法不同。

### 4. 多重继承冲突（菱形问题）

当一个类实现多个接口，且接口有同名 default 方法时，**必须显式重写解决冲突**：

```java
interface A {
    default void hello() { System.out.println("A"); }
}

interface B {
    default void hello() { System.out.println("B"); }
}

// ❌ 编译错误：不明确
class C implements A, B {}

// ✅ 方式一：重写方法
class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // 调用 A 的默认实现
    }
}

// ✅ 方式二：只实现其中一个
class C implements A, B {
    @Override
    public void hello() {
        B.super.hello();  // 调用 B 的默认实现
    }
}
```

### 5. default 方法 vs 抽象类

| 对比项 | default 方法 | 抽象类 |
|--------|-------------|--------|
| 实例状态 | 不能有实例字段（只能有 static final） | 可以有实例字段 |
| 构造方法 | 没有 | 有 |
| 多继承 | 可以实现多个接口 | 只能继承一个类 |
| 访问修饰符 | 隐式 public | 可以是 protected/private |

### 6. 关键规则

- default 方法可以被重写，重写时可以去掉 `default` 关键字
- 接口可以继承另一个接口，并重写其 default 方法
- `Object` 类的方法（`equals`/`hashCode`/`toString`）不能在接口中作为 default 方法（因为每个类都从 Object 继承了）

---

## 代码示例

```java
public interface Flyable {
    void fly();

    default void takeOff() {
        System.out.println("准备起飞...");
        fly();
    }

    static void checkWeather() {
        System.out.println("天气检查通过");
    }
}

public class Airplane implements Flyable {
    @Override
    public void fly() {
        System.out.println("飞机飞行中");
    }

    // takeOff() 继承了默认实现，可以不重写
}

// 使用
Flyable.checkWeather();     // 接口静态方法
Airplane plane = new Airplane();
plane.takeOff();            // 使用 default 方法
```

---

# Optional 类

## 核心结论

`Optional<T>` 是 JDK 8 引入的**容器类**，用于表示一个值可能存在也可能不存在。核心目的是**消除 NullPointerException**，以显式的方式提醒开发者处理空值情况。`Optional` 不应用于成员变量和方法参数，主要用于**方法返回值**。

---

## 深度解析

### 1. 创建 Optional

```java
// 创建非空 Optional
Optional<String> opt1 = Optional.of("hello");

// 创建可能为空的 Optional（推荐）
Optional<String> opt2 = Optional.ofNullable(null);  // Optional.empty
Optional<String> opt3 = Optional.ofNullable("hello");

// 创建空 Optional
Optional<String> empty = Optional.empty();

// ⚠️ of(null) 抛 NullPointerException
Optional<String> bad = Optional.of(null);
```

### 2. 判断与获取值

```java
Optional<String> opt = Optional.ofNullable(getName());

// 判断是否有值
opt.isPresent();    // boolean
opt.isEmpty();      // boolean（JDK 11）

// 获取值（危险！没有值时抛 NoSuchElementException）
opt.get();

// 获取值（带默认值）
opt.orElse("default");

// 获取值（带默认值，默认值延迟计算）
opt.orElseGet(() -> computeDefault());

// 获取值（无值时抛自定义异常）
opt.orElseThrow(() -> new BusinessException("名称不能为空"));
opt.orElseThrow();  // JDK 10，无值时抛 NoSuchElementException
```

### 3. 链式操作

```java
Optional<String> opt = Optional.ofNullable(user)
    .map(User::getName)          // 映射
    .filter(name -> name.length() > 3)  // 过滤
    .map(String::toUpperCase);   // 再映射

// map：转换值类型
Optional<Integer> length = opt.map(String::length);

// flatMap：避免嵌套 Optional
Optional<Optional<String>> nested = opt.map(this::findDetail);   // 嵌套
Optional<String> flat = opt.flatMap(this::findDetail);           // 扁平化
```

### 4. 消费值

```java
// ifPresent：有值时执行操作
opt.ifPresent(name -> System.out.println("Hello, " + name));

// ifPresentOrElse：有值和无值分别处理（JDK 9）
opt.ifPresentOrElse(
    name -> System.out.println("Hello, " + name),
    () -> System.out.println("Guest")
);
```

### 5. Stream 集成

```java
// 将 Optional 转为 Stream（JDK 9）
Stream<String> stream = opt.stream();  // 空时返回空 Stream

// 实际场景：过滤 Optional 列表
List<Optional<String>> opts = Arrays.asList(Optional.of("A"), Optional.empty(), Optional.of("B"));
List<String> result = opts.stream()
    .flatMap(Optional::stream)   // 展开为 Stream<String>
    .collect(Collectors.toList()); // ["A", "B"]
```

### 6. or() 方法（JDK 9）

```java
// 当 Optional 为空时，使用备选的 Optional
Optional<String> primary = Optional.ofNullable(findPrimary());
Optional<String> result = primary.or(() -> findBackup());
```

### 7. 使用规范

| 场景 | 建议 |
|------|------|
| 方法返回值 | ✅ 推荐，明确表示可能为空 |
| 成员变量 | ❌ 不推荐，增加序列化复杂度 |
| 方法参数 | ❌ 不推荐，增加调用复杂度 |
| 集合元素 | ❌ 不推荐，集合本身可以为空 |
| `optional.get()` | ❌ 尽量避免，应先判断或用 orElse |

---

## 代码示例

```java
// 典型用法：方法返回值
public Optional<User> findUserById(Long id) {
    return Optional.ofNullable(userRepository.findById(id));
}

// 调用链
String userName = findUserById(1L)
    .map(User::getName)
    .filter(name -> !name.isEmpty())
    .orElse("未知用户");

// 错误示范：Optional 作为成员变量
public class User {
    private Optional<String> nickname;  // ❌ 不推荐
}

// 正确方式
public class User {
    private String nickname;  // ✅ 可以为 null
}
```

---


---

# Lambda 表达式

## 核心结论

Lambda 表达式（JDK 8）是 Java 引入的第一个函数式编程特性，本质上是**匿名内部类的语法糖**，但底层通过 `invokedynamic` 字节码指令实现，由 JDK 在运行期动态生成类，避免了传统匿名内部类需要单独编译成 `.class` 文件的开销。Lambda 让代码更简洁，核心价值是**将行为作为参数传递**。

---

## 深度解析

### 1. 为什么需要 Lambda？

传统 Java 中传递行为只能通过匿名内部类，代码冗长：

```java
// JDK 8 之前：传递一个"行为"需要写一个匿名内部类
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
}).start();

// JDK 8 之后：Lambda 表达式，一行搞定
new Thread(() -> System.out.println("Hello")).start();
```

> **核心转变**：从"传递对象"到"传递行为"，这是 Java 从面向对象向函数式风格迈进的第一步。

### 2. 基本语法

```java
// 完整语法：(参数列表) -> { 方法体 }
(int a, int b) -> { return a + b; }

// 省略括号（只有一个参数）
x -> x * 2

// 省略花括号（只有一条语句，且该语句是返回值）
x -> x + 1

// 无参数
() -> System.out.println("Hello")

// 多条语句
(x, y) -> {
    int sum = x + y;
    return sum;
}
```

### 3. 函数式接口约束

Lambda 表达式必须关联到一个**函数式接口**（只有一个抽象方法的接口）。编译器自动推断类型：

```java
@FunctionalInterface  // 可选注解，约束只有一个抽象方法
interface MathOperation {
    int apply(int a, int b);
}

// Lambda 自动匹配 MathOperation
MathOperation add = (a, b) -> a + b;
MathOperation multiply = (a, b) -> a * b;

System.out.println(add.apply(2, 3));      // 5
System.out.println(multiply.apply(2, 3)); // 6
```

### 4. 底层原理：invokedynamic

这是面试高频深水区。Lambda 不是通过匿名内部类实现的，而是通过 `invokedynamic` 指令：

```java
// 源码
Runnable r = () -> System.out.println("lambda");
```

```java
// 编译后的字节码（关键部分）
INVOKEDYNAMIC "run()Ljava/lang/Runnable"  // JDK 动态生成类
    // BootstrapMethods:
    //   LAMBDA: () -> new Lambda$$1()
    // Lambda 运行时通过 LambdaMetafactory.metafactory() 动态生成实现类
```

**执行流程**：

```
源码：() -> System.out.println("hello")
       ↓
javac 编译：生成 invokedynamic 调用点
       ↓
首次执行时：JVM 调用 LambdaMetafactory.metafactory()
       ↓
运行时动态生成：lambda$1 implements Runnable
       ↓
缓存实现类：后续调用直接使用
```

> **与匿名内部类的本质区别**：匿名内部类在编译时就生成了 `.class` 文件（`OuterClass$1.class`），而 Lambda 在运行期通过 invokedynamic 动态生成，**不会产生额外的 `.class` 文件**。

### 5. 捕获机制

Lambda 可以捕获外部变量，但有严格限制：

```java
// ✅ 捕获局部变量（必须是 final 或 effectively final）
int base = 10;
Function<Integer, Integer> add = x -> x + base;  // OK，base 是 effectively final

// ❌ 不能修改捕获的变量（JDK 8 之前匿名内部类也不行）
int count = 0;
Runnable r = () -> count++;  // 编译错误！count 不是 final

// ✅ Lambda 外部可以修改 count
int count = 0;
Runnable r = () -> {
    System.out.println(count);  // 读取 OK
};
count = 10;  // 外部修改 OK
```

> **"effectively final"**：JDK 8 引入的概念——变量在初始化后不再修改，编译器视为 final，可以被 Lambda 捕获。

### 6. 方法引用

方法引用是 Lambda 的简写形式，分为四类：

```java
class Dog {
    String name;

    Dog(String name) { this.name = name; }

    static void sleep() { System.out.println("Sleeping..."); }

    void bark() { System.out.println(name + " barks"); }
}

// ① 静态方法引用：ClassName::staticMethod
Function<String, Integer> parser = Integer::parseInt;

// ② 实例方法引用（特定对象）：instance::instanceMethod
Dog dog = new Dog("旺财");
Callable<String> bark = dog::bark;

// ③ 实例方法引用（任意对象）：ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;
// 等价于：s -> s.toUpperCase()

// ④ 构造器引用：ClassName::new
Function<String, Dog> dogFactory = Dog::new;
// 等价于：name -> new Dog(name)
```

### 7. 变量作用域

```java
public class LambdaScope {
    int x = 10;

    void run() {
        int y = 5;  // effectively final

        // Lambda 内部可以访问类的实例字段
        Runnable r = () -> System.out.println(x + y);

        // Lambda 内部不能定义与外部同名的变量
        // int y = 20;  // ❌ 编译错误

        // Lambda 内部不能修改外部局部变量
        // y++;  // ❌ 编译错误
    }
}
```

### 8. Lambda 与匿名内部类的对比

| 对比项 | Lambda 表达式 | 匿名内部类 |
|--------|-------------|-----------|
| 关键字 | `->` | `new` + 方法重写 |
| 编译产物 | `invokedynamic` 运行时生成 | 编译时生成 `.class` 文件 |
| `this` 指向 | 指向外部类实例 | 指向匿名内部类实例 |
| 变量捕获 | 只能捕获 final / effectively final | 可以修改外部变量（JDK 8 前行为不同）|
| 性能 | 首次调用稍慢，后续无额外开销 | 每次 new 一个类 |
| 适用场景 | 函数式接口（单一抽象方法）| 需要定义字段、方法时的复杂场景 |

```java
// this 指向的区别
public class ThisDemo {
    Runnable lambda = () -> System.out.println(this);  // this = ThisDemo 实例
    Runnable anon = new Runnable() {
        @Override
        public void run() {
            System.out.println(this);  // this = 匿名内部类实例
        }
    };
}
```

---

## 代码示例

```java
// 典型使用场景：集合排序
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");

// JDK 7 之前
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// JDK 8 Lambda
Collections.sort(names, (a, b) -> a.compareTo(b));

// JDK 8+ 方法引用 + Comparator 组合
names.sort(String::compareTo);

// 典型使用场景：事件监听
button.setOnClickListener(e -> {
    // Lambda 处理事件
    System.out.println("按钮点击：" + e.getActionCommand());
});

// 典型使用场景：Stream 链式处理（与 Stream API 结合）
List<String> result = names.stream()
    .filter(s -> s.startsWith("A"))    // Lambda 作为Predicate
    .map(String::toUpperCase)           // 方法引用
    .collect(Collectors.toList());
```

---

# Stream API

## 核心结论

Stream API（JDK 8）是 Java 对集合操作的重大升级，提供**声明式**的数据处理方式（过滤→映射→聚合），与 Iterator 的区别在于 Stream 不会修改底层数据源（惰性求值），且只消费一次（不可重复遍历）。Stream 不是数据结构，不存储数据，是数据源的视图。

> **与 `10_基础/11_Stream API` 模块的关系**：本节是概述与面试要点，详细 API 操作请参考 Stream API 模块。

---

## 深度解析

### 1. 创建 Stream

```java
// 集合转 Stream
Stream<String> s1 = list.stream();
Stream<String> s2 = set.stream();

// 数组转 Stream
Stream<String> s3 = Arrays.stream(array);

// Stream.of 静态工厂
Stream<String> s4 = Stream.of("A", "B", "C");

// 无限 Stream（配合 limit）
Stream<Integer> s5 = Stream.iterate(0, n -> n + 2).limit(10);
Stream<Double> s6 = Stream.generate(Math::random).limit(5);

// 并行 Stream
Stream<String> parallel = list.parallelStream();
```

### 2. 中间操作（惰性）

中间操作返回 Stream，不会立即执行，只有遇到终端操作时才触发计算：

```java
list.stream()
    .filter(s -> s.length() > 3)  // 惰性：只记录这个操作
    .map(String::toUpperCase)       // 惰性：只记录这个操作
    .distinct()                     // 惰性
    .sorted()                       // 惰性
    .limit(10);                     // 惰性（还没执行！）
    // .count() 终端操作！从这里开始才真正执行
```

### 3. 终端操作（触发求值）

```java
// 聚合
long count = list.stream().filter(s -> s.length() > 3).count();
Optional<String> first = list.stream().findFirst();
Optional<String> max = list.stream().max(Comparator.comparingInt(String::length));

// 收集
List<String> list2 = stream.collect(Collectors.toList());
Set<String> set2 = stream.collect(Collectors.toSet());
Map<String, Integer> map = stream.collect(Collectors.toMap(s -> s, String::length));
String joined = stream.collect(Collectors.joining(", "));

// 遍历
stream.forEach(System.out::println);

// 匹配
boolean anyMatch = list.stream().anyMatch(s -> s.startsWith("A"));
boolean allMatch = list.stream().allMatch(s -> s.length() > 0);
```

### 4. 常见使用模式

```java
// 分组
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity));

// 分区（按条件分为两组）
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() > 18));

// 多级分组
Map<String, Map<String, List<Person>>> grouped = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
        Collectors.groupingBy(Person::getGender)));

// 统计
IntSummaryStatistics stats = people.stream()
    .collect(Collectors.summarizingInt(Person::getAge));
System.out.println("平均年龄：" + stats.getAverage());
```

### 5. 并行 Stream 的坑

并行流底层使用 `ForkJoinPool.commonPool()`，但不是万能的：

```java
// ❌ 错误场景：并行流 + 有状态操作
List<Integer> result = list.parallelStream()
    .map(x -> x + 1)
    .sorted()  // sorted() 是有状态操作，需要全部数据收集后再排序
    .collect(Collectors.toList());

// ❌ 错误场景：并行流 + 非线程安全的数据结构
ArrayList<String> unsafe = new ArrayList<>();
list.parallelStream()
    .forEach(unsafe::add);  // 不安全！并行写入 ArrayList
// ✅ 正确做法：使用线程安全的收集器
List<String> safe = list.parallelStream()
    .collect(Collectors.toList());

// ❌ 错误场景：小数据量用并行流反而更慢
list.parallelStream()  // 只有10个元素，并行开销 > 收益
    .map(String::toUpperCase)
    .count();
```

### 6. 性能考虑

| 场景 | 建议 |
|------|------|
| 大数据量（> 10000）| 并行流可显著提升性能 |
| 小数据量（< 100）| 串行流，避免线程开销 |
| 有状态中间操作（sorted/distinct）| 慎用并行流，性能退化严重 |
| I/O 密集型 | 并行流效果明显 |
| 有副作用的操作 | 避免使用并行流，结果不可预测 |

---

## 代码示例

```java
// 实际场景：订单报表统计
List<Order> orders = getOrders();

Map<String, Long> cityOrderCount = orders.stream()
    .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
    .collect(Collectors.groupingBy(Order::getCity, Collectors.counting()));

Map<String, BigDecimal> cityRevenue = orders.stream()
    .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
    .collect(Collectors.groupingBy(Order::getCity,
        Collectors.reducing(BigDecimal.ZERO, Order::getAmount, BigDecimal::add)));

// 实际场景：多条件筛选 + 分页
List<Product> page = products.stream()
    .filter(p -> p.getPrice() > 100)
    .filter(p -> "电子产品".equals(p.getCategory()))
    .sorted(Comparator.comparing(Product::getSalesCount).reversed())
    .skip((pageNum - 1) * pageSize)
    .limit(pageSize)
    .collect(Collectors.toList());
```

---

## 关联知识点

