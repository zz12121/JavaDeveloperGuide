---
title: JDK 21新特性
tags:
  - Java/新特性
  - 问答
  - 原理型
module: 13_Java新特性
created: 2026-04-24
---

# Virtual Threads 虚拟线程

## Q1：什么是虚拟线程？它解决了什么问题？

**A**：虚拟线程是 JDK 21 正式引入的轻量级线程实现（Project Loom），由 JVM 而非操作系统调度。

**解决的核心问题**：传统平台线程与 OS 线程 1:1 绑定，每个线程约占 1MB 内存，服务器最多同时维持数千线程。高并发 IO 场景（如每个请求一个线程）资源消耗大。

**虚拟线程的优势**：
- 创建成本极低（KB 级别），可创建数百万个
- 阻塞时自动挂起，不占用底层平台线程
- 编程模型与平台线程完全相同（同步风格），无需改变代码习惯

```java
// 使用方式与普通线程几乎一样
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        Thread.sleep(Duration.ofMillis(100)); // 阻塞不占平台线程
    });
}
```

---

## Q2：虚拟线程与平台线程有什么区别？

**A**：

| 对比维度 | 平台线程 | 虚拟线程 |
|---------|---------|---------|
| 与 OS 关系 | 1:1 映射 OS 线程 | M:N 映射到平台线程（载体线程） |
| 创建成本 | 高（~1MB 栈） | 极低（~几 KB） |
| 数量上限 | 数千 | 数百万 |
| 阻塞行为 | 阻塞 OS 线程 | 挂起，释放载体线程 |
| 适用场景 | CPU 密集型 | IO 密集型（网络、数据库） |

**一句话**：虚拟线程专为 IO 密集型场景设计，让你用同步风格写出异步性能。

---

## Q3：虚拟线程有什么坑？`synchronized` 会有问题吗？

**A**：有，这是最重要的陷阱。

**JDK 21 中，`synchronized` 块会"pin"住载体线程**：虚拟线程在执行 `synchronized` 代码时遇到阻塞，无法 unmount，会一直占用底层平台线程，退化为平台线程的表现。

```java
// 问题写法：synchronized 中有 IO 阻塞
synchronized (lock) {
    data = db.query();  // 会 pin 住载体线程！
}

// 推荐写法：改用 ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    data = db.query();  // 虚拟线程可以正常 unmount
} finally {
    lock.unlock();
}
```

**其他注意点**：
- Native 方法调用同样会 pin 住载体线程
- 不要用线程池管理虚拟线程（池化是补偿手段，虚拟线程直接 new 即可）
- CPU 密集型任务不适合虚拟线程

---

## Q4：虚拟线程适合替代 WebFlux / 响应式编程吗？

**A**：在大多数场景下可以。

**可以替代的场景**：IO 密集型服务（HTTP 调用、数据库查询），虚拟线程可以达到相近的吞吐量，且代码更简单易懂。

**不适合替代的场景**：
- 背压（Backpressure）控制：响应式编程可精细控制数据流速率，虚拟线程不具备
- 函数式流处理管道：Reactor 的操作符链（map/flatMap/filter）更适合流处理
- 已有大量响应式代码的系统：迁移成本高

**Spring Boot 建议**：Spring Boot 3.2+ 可通过 `spring.threads.virtual.enabled=true` 一键启用，适合新项目优先选择虚拟线程方案。

---

## Q5：如何创建虚拟线程？有几种方式？

**A**：三种常用方式：

```java
// 方式一：Thread.ofVirtual()
Thread vt = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> System.out.println("Hello"));
vt.join();

// 方式二：ExecutorService（最常用）
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> doWork());
}

// 方式三：Thread.startVirtualThread()（最简短）
Thread.startVirtualThread(() -> System.out.println("快速启动"));
```

生产环境推荐**方式二**，配合 try-with-resources 自动关闭，符合线程池管理规范（虽然底层不复用线程）。

---

# Pattern Matching for switch 模式匹配 Switch

## Q6：什么是 Switch 模式匹配？与传统 Switch 有什么区别？

**A**：Switch 模式匹配（JDK 21 正式，JEP 441）允许 `switch` 对任意类型进行类型匹配，并自动绑定变量，还支持 `when` 守卫条件和 `null` 处理。

```java
// 传统 switch（只支持基本类型/String/枚举）
// 类型判断只能靠 if-instanceof 链

// Switch 模式匹配
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "正整数: " + i;
        case Integer i            -> "非正整数: " + i;
        case String s             -> "字符串: " + s;
        case null                 -> "null";
        default                   -> "其他";
    };
}
```

**核心优势**：
1. 彻底替代 `if-instanceof` 链，代码更简洁
2. `when` 子句实现条件过滤，替代嵌套 if
3. 可直接处理 `null`，传统 switch 传入 null 会抛 NPE
4. 配合密封类，编译器自动检查穷举性（无需 default）

---

## Q7：Switch 模式匹配中的 `when` 子句是什么？

**A**：`when` 是守卫模式（Guarded Pattern），在类型匹配成功后，**追加额外的布尔条件**，只有条件也为 true 时才匹配该分支。

```java
// when 子句示例
String classify(Number n) {
    return switch (n) {
        case Integer i when i < 0   -> "负整数";
        case Integer i when i == 0  -> "零";
        case Integer i              -> "正整数";
        case Double d when d.isNaN()-> "非数字";
        case Double d               -> "浮点数: " + d;
        default                     -> "其他数字";
    };
}
```

匹配规则：从上到下顺序匹配，第一个类型和 `when` 条件都满足的分支执行。

---

## Q8：Switch 模式匹配与密封类配合有什么优势？

**A**：密封类限定了子类的范围，配合 Switch 模式匹配，**编译器可以验证是否穷举了所有子类型**，无需 `default` 分支，新增子类时编译器会强制报错提醒处理。

```java
sealed interface Result permits Success, Failure, Loading {}
record Success(String data) implements Result {}
record Failure(String msg)  implements Result {}
record Loading()            implements Result {}

String render(Result result) {
    return switch (result) {
        case Success(var data) -> "成功: " + data;
        case Failure(var msg)  -> "失败: " + msg;
        case Loading _         -> "加载中...";
        // 无 default：编译器保证已穷举，如果 Result 新增子类会报编译错误
    };
}
```

---

# Record Patterns 记录模式

## Q9：Record Pattern 是什么？如何在 instanceof 中使用？

**A**：Record Pattern（JDK 21 正式，JEP 440）允许在 `instanceof` 和 `switch` 中**直接解构 Record 的组件字段**，无需手动调用访问器。

```java
record Point(int x, int y) {}

// 传统写法
if (obj instanceof Point p) {
    int x = p.x();
    int y = p.y();
    System.out.println(x + ", " + y);
}

// Record Pattern（直接解构）
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);  // 直接用 x, y
}
```

**嵌套解构**：
```java
record Coordinates(double lat, double lon) {}
record Location(String name, Coordinates coords) {}

// 一次解构两层
if (obj instanceof Location(String name, Coordinates(double lat, double lon))) {
    System.out.printf("%s: %.4f, %.4f%n", name, lat, lon);
}
```

---

# Sequenced Collections 序列集合

## Q10：JDK 21 的 Sequenced Collections 解决了什么历史问题？

**A**：解决了不同集合类型访问首尾元素时 **API 不一致**的问题。

```java
// 历史问题：每种集合获取首尾元素方式各不同
list.get(0)                    // List 第一个
list.get(list.size() - 1)     // List 最后一个（容易写错）
deque.getFirst()               // Deque 第一个
sortedSet.first()              // SortedSet 第一个

// JDK 21 统一 API（SequencedCollection）
collection.getFirst()          // 任何有序集合的第一个
collection.getLast()           // 任何有序集合的最后一个
collection.reversed()          // 返回反转视图
```

**新增三个接口**：
- `SequencedCollection`：有序集合的首尾操作
- `SequencedSet`：无重复的有序集合
- `SequencedMap`：有序 Map 的首尾 Entry 操作

`List`、`Deque`、`LinkedHashSet`、`LinkedHashMap`、`TreeSet`、`TreeMap` 均已实现相应接口。

---

## Q11：`SequencedCollection.reversed()` 是创建新集合吗？

**A**：**不是**，`reversed()` 返回的是**原集合的反转视图**（Reverse View），不复制元素。对原集合的修改会反映在反转视图中，反之亦然。

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
List<String> rev = list.reversed();  // 视图，非拷贝

System.out.println(rev);  // [c, b, a]

list.add("d");
System.out.println(rev);  // [d, c, b, a]  — 同步变化
```

---

## Q12：JDK 21 是 LTS 版本吗？主要包含哪些重要特性？

**A**：**是的，JDK 21 是继 JDK 17 之后的 LTS（长期支持）版本**，发布于 2023 年 9 月。

**正式发布的重要特性**：

| 特性 | JEP | 说明 |
|------|-----|------|
| Virtual Threads | 444 | 虚拟线程，百万级并发 IO |
| Pattern Matching for switch | 441 | Switch 模式匹配正式版 |
| Record Patterns | 440 | 记录模式解构正式版 |
| Sequenced Collections | 431 | 统一首尾操作接口 |
| Unnamed Classes and Instance Main | 445 | 预览：简化小程序入口 |

**对比 JDK 17**：JDK 21 是 Java 并发编程范式的重大转折点，虚拟线程从根本上改变了高并发应用的开发方式。

---

## Q13：使用虚拟线程是否需要修改大量现有代码？

**A**：基本不需要大改。虚拟线程的 API 与平台线程完全兼容，主要的改动点是：

1. **替换线程池**：`Executors.newFixedThreadPool(n)` → `Executors.newVirtualThreadPerTaskExecutor()`
2. **检查 synchronized 块**：含 IO 阻塞的 `synchronized` 块改为 `ReentrantLock`
3. **Spring Boot 项目**：只需加一行配置 `spring.threads.virtual.enabled=true`

**不需要改动**：ThreadLocal、try-catch、业务逻辑代码、框架库代码（Spring、MyBatis 等主流框架已支持）。

---

# ScopedValue 与 Structured Concurrency

## Q14：ScopedValue 是什么？和 ThreadLocal 有什么区别？

**A**：`ScopedValue<T>`（JEP 446，JDK 21 正式）是线程隔离值的新方案，用来替代 `ThreadLocal`。

**核心区别**：

| 对比项 | ThreadLocal | ScopedValue |
|--------|-----------|-------------|
| 继承性 | 子线程可继承（InheritableThreadLocal）| ❌ 不可继承，虚拟线程也不继承 |
| 清理机制 | 需手动 `remove()`，容易遗漏导致内存泄漏 | 自动清理（离开作用域即失效）|
| 内存泄漏风险 | 有（线程池复用场景）| 无 |
| 适用场景 | 通用 | 虚拟线程 + 结构化并发（配合更好）|

```java
// ThreadLocal：需要手动清理，容易遗漏
ThreadLocal<User> currentUser = new ThreadLocal<>();
try {
    currentUser.set(user);
    doWork();
} finally {
    currentUser.remove();  // 必须手动，否则内存泄漏
}

// ScopedValue：作用域自动管理
ScopedValue<User> currentUser = ScopedValue.newInstance();
ScopedValue.where(currentUser, user)
    .run(() -> doWork());  // 作用域结束自动清理，无需手动
```

**推荐迁移路径**：新增代码用 `ScopedValue`，已有 `ThreadLocal` 代码可继续使用，JDK 21 仍完全兼容。

---

## Q15：StructuredTaskScope 是什么？解决了什么问题？

**A**：`StructuredTaskScope<T>`（JEP 453，JDK 21 正式）将并发任务的管理结构化，保证子任务的生命周期不超过父任务，从根本上解决线程逃逸问题。

**解决的问题**：传统 `ExecutorService` 中，如果父任务抛出异常，`shutdown()` 可能永远不会执行，导致资源泄漏。`StructuredTaskScope` 通过 try-with-resources 确保作用域退出时自动清理。

```java
// 传统方式：异常可能导致资源泄漏
try {
    executor.submit(task1);
    executor.submit(task2);
    executor.shutdown();
} catch (Exception e) {
    executor.shutdown();  // 如果这里抛异常，shutdown 不会执行
}

// 结构化并发：try-with-resources 保证清理
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> f1 = scope.fork(() -> queryDatabase());
    Future<String> f2 = scope.fork(() -> callService());

    scope.join();  // 等待所有完成

    if (scope.failed()) {
        throw new ExecutionException(scope.exceptionOf(f1));
    }
    return merge(f1.resultNow(), f2.resultNow());
}
// 作用域退出：自动取消未完成任务，清理资源
```

**两种策略**：
- `ShutdownOnFailure`：任一任务失败 → 取消其他 → 汇总失败
- `ShutdownOnSuccess`：任一任务成功 → 取消其他 → 返回成功结果

---

## Q16：String Templates 为什么被撤回了？

**A**：String Templates（JEP 430）在 JDK 21 预览后，在 JDK 24 被正式撤回（Removed）。主要原因是**安全性设计缺陷**：内置的 `STR` 处理器理论上仍可能存在注入风险（如 SQL 注入），而自定义模板处理器虽然可以增强安全性，但门槛较高，社区对整体 API 设计方向存在争议。

**后续方向**：JDK 团队正在重新设计该特性，未来可能以更安全的 API 重新引入。生产环境建议继续使用 `String.format()`、`MessageFormat` 或第三方模板库（Mustache、StringTemplate 等）。

---

## Q17：JDK 17 和 JDK 21 的 LTS 版本应该如何选择？

**A**：**新项目建议直接用 JDK 21**，JDK 17 可以继续维护。

| 对比 | JDK 17（LTS，2021）| JDK 21（LTS，2023）|
|------|---------------------|---------------------|
| 虚拟线程 | ❌ 不支持 | ✅ JDK 21 正式支持 |
| 结构化并发 | ❌ | ✅ |
| ScopedValue | ❌ | ✅ |
| Switch 模式匹配 | ❌ | ✅ 正式支持 |
| Record Patterns | ❌ | ✅ 正式支持 |
| 生态兼容性 | Spring Boot 6.x / Spring Framework 6.x | Spring Boot 3.x |
| 建议 | 现有项目维护 | 新项目首选 |

> **关键区别**：JDK 21 是 Java 并发编程范式的转折点——虚拟线程从根本上改变了高并发应用的开发方式，JDK 17 则没有如此重大的范式变更。
