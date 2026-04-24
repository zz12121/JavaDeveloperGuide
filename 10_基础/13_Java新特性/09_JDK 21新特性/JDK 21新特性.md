---
title: JDK 21新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-24
---

# Virtual Threads 虚拟线程（JDK 21 正式）

## 核心结论

Virtual Threads（虚拟线程）是 JDK 21 正式引入的轻量级线程实现（Project Loom），由 JVM 而非操作系统调度。虚拟线程在阻塞时不占用平台线程（载体线程），可以创建**数百万**级别的线程，从根本上解决传统"一请求一线程"模型在高并发场景下的资源瓶颈。

---

## 深度解析

### 1. 背景与动机

传统平台线程（Platform Thread）与 OS 线程 1:1 绑定，创建成本约 1MB 栈空间 + 上下文切换开销，典型服务器能维持的并发线程数约在数千级别。

常见应对方案是异步编程（CompletableFuture、Reactor、WebFlux），但带来代码复杂性高、调试困难、与同步生态割裂等问题。

虚拟线程目标：**写同步风格的代码，获得异步级别的吞吐量**。

### 2. 核心概念

```
Platform Thread（平台线程）= OS Thread（1:1）
Virtual Thread（虚拟线程）= JVM 管理，M:N 映射到 Platform Thread
```

| 对比维度 | 平台线程 | 虚拟线程 |
|---------|---------|---------|
| 创建成本 | 高（~1MB 栈） | 极低（~几 KB） |
| 数量上限 | 数千 | 数百万 |
| 调度方 | OS | JVM（ForkJoinPool） |
| 阻塞行为 | 阻塞 OS 线程 | 挂起并释放载体线程 |
| 编程模型 | 同步 | **同步**（保持不变） |

### 3. 基本用法

```java
// 方式一：Thread.ofVirtual()
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("运行在虚拟线程：" + Thread.currentThread());
});
vt.join();

// 方式二：newVirtualThreadPerTaskExecutor()
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            // 阻塞 IO，虚拟线程会挂起，不占平台线程
            Thread.sleep(Duration.ofMillis(100));
            return "done";
        });
    }
} // AutoCloseable，自动关闭
```

### 4. 虚拟线程调度原理

```
虚拟线程发起阻塞调用（IO/sleep/锁等待）
       ↓
JVM 检测到阻塞点（mount/unmount 机制）
       ↓
将虚拟线程从载体线程卸载（unmount），状态保存在堆上
       ↓
载体线程（ForkJoinPool 工作线程）可继续执行其他虚拟线程
       ↓
阻塞完成后，虚拟线程重新挂载（mount）到某个可用载体线程继续执行
```

### 5. 注意事项与限制

| 场景 | 说明 |
|------|------|
| synchronized 块 | **会 pin 住载体线程**（JDK 21 中 synchronized 不支持 unmount），应改用 `ReentrantLock` |
| native 方法调用 | 同样会 pin 住载体线程 |
| ThreadLocal | 虚拟线程支持 ThreadLocal，但数量极多时内存开销需注意，推荐用 `ScopedValue`（孵化中） |
| 线程池 | **不要用线程池管理虚拟线程**，每个任务都 `new` 一个即可，池化是平台线程时代的补偿手段 |
| CPU 密集型任务 | 虚拟线程没有优势，仍需限制并发数 |

> **面试高频陷阱**：虚拟线程遇到 `synchronized` 会 pin 住载体线程，JDK 21 中应将同步块替换为 `ReentrantLock` 以发挥虚拟线程最大效益。

### 6. Spring Boot 启用虚拟线程

```yaml
# Spring Boot 3.2+ application.yml
spring:
  threads:
    virtual:
      enabled: true
```

```java
// 或手动配置 Tomcat 使用虚拟线程
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadCustomizer() {
    return handler -> handler.setExecutor(
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

## 代码示例：百万并发 HTTP 请求

```java
// 模拟 100 万并发 IO 场景
public static void main(String[] args) throws InterruptedException {
    int taskCount = 1_000_000;
    CountDownLatch latch = new CountDownLatch(taskCount);

    try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < taskCount; i++) {
            executor.submit(() -> {
                try {
                    // 模拟 IO 阻塞（数据库查询/HTTP 调用）
                    Thread.sleep(Duration.ofMillis(500));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            });
        }
    }

    latch.await();
    System.out.println("百万任务完成");
}
```

## 关联知识点

---

# Pattern Matching for switch 模式匹配 Switch（JDK 21 正式）

## 核心结论

Switch 模式匹配（JEP 441）在 JDK 21 正式发布，允许 `switch` 表达式和语句对任意类型进行**类型模式匹配**，支持 `null` 处理、守卫模式（`when` 子句）、以及与密封类配合时的**穷举性检查**。彻底替代 `instanceof` 链式 if-else。

---

## 深度解析

### 1. 基本语法：类型模式

```java
// JDK 21 之前（丑陋的 instanceof 链）
String describe(Object obj) {
    if (obj instanceof Integer i) return "int " + i;
    else if (obj instanceof String s) return "string " + s;
    else if (obj instanceof Long l) return "long " + l;
    else return "unknown";
}

// JDK 21 Switch 模式匹配（简洁、类型安全）
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int " + i;
        case String s  -> "string " + s;
        case Long l    -> "long " + l;
        default        -> "unknown";
    };
}
```

### 2. 守卫模式（`when` 子句）

```java
String classify(Object obj) {
    return switch (obj) {
        case Integer i when i > 0  -> "正整数: " + i;
        case Integer i when i < 0  -> "负整数: " + i;
        case Integer i             -> "零";
        case String s when s.isEmpty() -> "空字符串";
        case String s              -> "字符串: " + s;
        default                    -> "其他类型";
    };
}
```

### 3. 处理 null

```java
// JDK 21 之前，switch 传入 null 会抛 NullPointerException
// JDK 21 可以直接在 case 中处理 null
String handle(String s) {
    return switch (s) {
        case null      -> "空值";
        case "hello"   -> "你好";
        default        -> "其他: " + s;
    };
}
```

### 4. 与密封类配合：穷举性检查

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double base, double height) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c      -> Math.PI * c.radius() * c.radius();
        case Rectangle r   -> r.w() * r.h();
        case Triangle t    -> 0.5 * t.base() * t.height();
        // 无需 default：编译器保证 sealed 类型已全部覆盖
    };
}
```

### 5. 对比总结

| 特性 | 传统 if-instanceof 链 | Switch 模式匹配 |
|------|----------------------|----------------|
| 类型提取 | 手动 `instanceof` + 强转 | 自动绑定变量 |
| null 处理 | 需手动判断 | `case null` 直接处理 |
| 穷举检查 | 无 | 密封类场景编译器保证 |
| 守卫条件 | if-else 嵌套 | `when` 子句内联 |
| 代码可读性 | 差 | 好 |

## 代码示例

```java
// 实战：处理 JSON 类型系统（简化版）
sealed interface JsonValue
    permits JsonNull, JsonBool, JsonNumber, JsonString, JsonArray, JsonObject {}

record JsonNull()                     implements JsonValue {}
record JsonBool(boolean value)        implements JsonValue {}
record JsonNumber(double value)       implements JsonValue {}
record JsonString(String value)       implements JsonValue {}
record JsonArray(List<JsonValue> list) implements JsonValue {}
record JsonObject(Map<String, JsonValue> map) implements JsonValue {}

String prettyPrint(JsonValue v) {
    return switch (v) {
        case JsonNull _                    -> "null";
        case JsonBool(var b)               -> String.valueOf(b);
        case JsonNumber(var n) when n % 1 == 0 -> String.valueOf((long) n);
        case JsonNumber(var n)             -> String.valueOf(n);
        case JsonString(var s)             -> "\"" + s + "\"";
        case JsonArray(var list)           -> list.stream()
            .map(this::prettyPrint).collect(Collectors.joining(", ", "[", "]"));
        case JsonObject(var map)           -> map.entrySet().stream()
            .map(e -> "\"" + e.getKey() + "\": " + prettyPrint(e.getValue()))
            .collect(Collectors.joining(", ", "{", "}"));
    };
}
```

## 关联知识点

---

# Record Patterns 记录模式（JDK 21 正式）

## 核心结论

Record Patterns（JEP 440）在 JDK 21 正式发布，允许在 `instanceof` 和 `switch` 中**解构 Record 的组件**，直接绑定组件变量，无需手动调用访问器方法，实现嵌套解构和深度模式匹配。

---

## 深度解析

### 1. 基本解构

```java
record Point(int x, int y) {}

// JDK 21 之前
if (obj instanceof Point p) {
    int x = p.x();
    int y = p.y();
    System.out.println(x + ", " + y);
}

// JDK 21 Record Pattern（直接解构）
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);  // 直接使用 x, y
}
```

### 2. 嵌套解构

```java
record Coordinates(double lat, double lon) {}
record Location(String name, Coordinates coords) {}

void printCoords(Object obj) {
    // 嵌套解构：一次性提取深层字段
    if (obj instanceof Location(String name, Coordinates(double lat, double lon))) {
        System.out.printf("%s: (%.4f, %.4f)%n", name, lat, lon);
    }
}
```

### 3. 在 Switch 中使用记录模式

```java
sealed interface Expr permits Num, Add, Mul {}
record Num(int value) implements Expr {}
record Add(Expr left, Expr right) implements Expr {}
record Mul(Expr left, Expr right) implements Expr {}

int eval(Expr expr) {
    return switch (expr) {
        case Num(int v)            -> v;
        case Add(Expr l, Expr r)   -> eval(l) + eval(r);
        case Mul(Expr l, Expr r)   -> eval(l) * eval(r);
    };
}
```

### 4. 与 `var` 配合（类型推断）

```java
record Pair<A, B>(A first, B second) {}

if (obj instanceof Pair(var first, var second)) {
    // 编译器推断 first 和 second 的类型
    System.out.println(first + " -> " + second);
}
```

## 关联知识点

---

# Sequenced Collections 序列集合（JDK 21 正式）

## 核心结论

Sequenced Collections（JEP 431）在 JDK 21 引入三个新接口：`SequencedCollection`、`SequencedSet`、`SequencedMap`，为有序集合提供**统一的首尾元素操作 API**，解决不同集合类型访问首尾元素时 API 不一致的历史问题。

---

## 深度解析

### 1. 历史问题

```java
// 获取 List 第一个元素（不同集合方式各不同！）
list.get(0)                    // List
deque.getFirst()               // Deque
sortedSet.first()              // SortedSet

// 获取最后一个元素
list.get(list.size() - 1)     // List（容易出错）
deque.getLast()                // Deque
sortedSet.last()               // SortedSet
```

### 2. 三个新接口

```java
// SequencedCollection：有序集合（有明确定义的遭遇顺序）
interface SequencedCollection<E> extends Collection<E> {
    SequencedCollection<E> reversed();  // 返回反转视图
    void addFirst(E e);
    void addLast(E e);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
}

// SequencedSet：无重复元素的有序集合
interface SequencedSet<E> extends SequencedCollection<E>, Set<E> {
    SequencedSet<E> reversed();
}

// SequencedMap：有序 Map
interface SequencedMap<K, V> extends Map<K, V> {
    SequencedMap<K, V> reversed();
    Map.Entry<K, V> firstEntry();
    Map.Entry<K, V> lastEntry();
    Map.Entry<K, V> pollFirstEntry();
    Map.Entry<K, V> pollLastEntry();
    void putFirst(K k, V v);
    void putLast(K k, V v);
}
```

### 3. 统一 API 对比

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// JDK 21 统一方式
String first = list.getFirst();   // "a"
String last  = list.getLast();    // "c"
list.addFirst("z");               // ["z", "a", "b", "c"]
list.addLast("x");                // ["z", "a", "b", "c", "x"]

// 反转视图（不创建新集合）
List<String> reversed = list.reversed();  // ["x", "c", "b", "a", "z"]

// LinkedHashMap 也支持
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("one", 1);
map.put("two", 2);
map.put("three", 3);

Map.Entry<String, Integer> firstEntry = map.firstEntry(); // one=1
Map.Entry<String, Integer> lastEntry  = map.lastEntry();  // three=3
```

### 4. 集合接口继承关系变化

```
Collection
└── SequencedCollection（新增）
    ├── List（ArrayList, LinkedList）
    ├── Deque（ArrayDeque, LinkedList）
    └── SequencedSet（新增）
        ├── LinkedHashSet
        └── SortedSet → NavigableSet（TreeSet）

Map
└── SequencedMap（新增）
    ├── LinkedHashMap
    └── SortedMap → NavigableMap（TreeMap）
```

## 关联知识点

---

# String Templates 字符串模板（JDK 21 预览）

## 核心结论

String Templates（JEP 430，JDK 21 预览）提供了安全的字符串插值机制，通过**模板处理器**（Template Processor）对插值内容进行验证和转换，相比 `+` 拼接和 `String.format()` 更安全、更易读。（注意：JDK 23 已撤回该特性，重新设计中）

---

## 深度解析

### 1. 基本语法

```java
String name = "张三";
int age = 25;

// 传统方式
String s1 = "姓名：" + name + "，年龄：" + age;
String s2 = String.format("姓名：%s，年龄：%d", name, age);

// String Templates（JDK 21 预览）
// STR 是内置模板处理器
String s3 = STR."姓名：\{name}，年龄：\{age}";
```

### 2. 内置模板处理器

| 处理器 | 说明 |
|--------|------|
| `STR` | 直接插值，返回 `String` |
| `FMT` | 支持格式化说明符（类似 `printf`） |
| `RAW` | 返回 `StringTemplate` 对象，不立即求值 |

```java
double price = 9.9;

// STR：简单插值
String s1 = STR."价格：\{price}";             // "价格：9.9"

// FMT：格式化插值
String s2 = FMT."价格：%.2f\{price}";          // "价格：9.90"
```

### 3. 安全性：防 SQL 注入

```java
// 自定义模板处理器，防止 SQL 注入
String userInput = "'; DROP TABLE users; --";

// 危险！直接拼接
String sql = "SELECT * FROM users WHERE name = '" + userInput + "'";

// 安全：自定义处理器可对 \{} 内容进行转义或参数化
PreparedStatement stmt = SQL."SELECT * FROM users WHERE name = \{userInput}";
// 自定义 SQL 处理器将插值转换为 PreparedStatement 参数，而非字符串拼接
```

> **注意**：String Templates 在 JDK 21 为预览特性，JDK 22/23 中继续预览，目前该特性已被撤回，重新设计中。生产环境请谨慎使用。

## 关联知识点

---

# Unnamed Patterns and Variables 无名模式与变量（JDK 21 预览）

## 核心结论

Unnamed Patterns（JEP 443，JDK 21 预览，JDK 22 正式）使用下划线 `_` 表示不需要使用的模式变量，提升代码可读性，明确表达"此处有意忽略"的语义。

---

## 深度解析

### 1. 无名变量

```java
// JDK 21 之前，声明但不使用的变量
try {
    int result = compute();
} catch (Exception e) {  // e 从未使用，但必须声明
    log.warn("计算失败");
}

// JDK 21 无名变量
try {
    int result = compute();
} catch (Exception _) {  // _ 明确表示"故意不使用"
    log.warn("计算失败");
}
```

### 2. 无名模式

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}

// 只关心是否是 Circle，不关心其内容
if (shape instanceof Circle _) {
    System.out.println("是圆形");
}

// Switch 中忽略不关心的组件
String describe(Shape shape) {
    return switch (shape) {
        case Circle(var r)    -> "圆，半径 " + r;
        case Rectangle(_, _) -> "矩形";  // 不关心具体尺寸
        case Triangle(_, _)  -> "三角形";
    };
}
```

### 3. 循环中使用

```java
List<String> items = List.of("a", "b", "c");

// 只关心执行次数，不关心元素值
for (var _ : items) {
    count++;
}
```

## 关联知识点
