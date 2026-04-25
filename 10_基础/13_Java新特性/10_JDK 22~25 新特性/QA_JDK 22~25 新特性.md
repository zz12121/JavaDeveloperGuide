---
title: JDK 22~25 新特性
tags:
  - Java
  - 新特性
  - 问答
module: 13_Java新特性
created: 2026-04-25
---

# JDK 22~25 新特性

## Q1：JDK 22~25 中有哪些从预览版升级为正式版的重大特性？

**A：**
JDK 22~25 是近年来 Java 正式化特性最多的版本周期，以下是主要升级为正式版的特性：

| 特性 | 预览版本 | 正式版本 |
| --- | --- | --- |
| **Stream Gatherers** | JDK 23 预览 → JDK 24 第三预览 | **JDK 24 正式** |
| **Foreign Function & Memory API** | JDK 22 第三预览 | **JDK 24 正式** |
| **Scoped Values** | JDK 21 第一预览 | **JDK 24 正式** |
| **Class-File API** | JDK 22 第一预览 | **JDK 24 正式** |
| **压缩字符串** | JDK 24 预览 | **JDK 25 正式** |
| **Structured Concurrency** | JDK 21 第一预览 → JDK 23 第二预览 | **JDK 25 正式** |
| **Primitive Patterns in instanceof** | JDK 23 预览 | **JDK 25 正式** |
| **Import Declarations** | JDK 21 预览 | **JDK 25 正式** |

**面试重点**：Stream Gatherers 和 FFM API 是 JDK 24 的两大核心特性，建议重点掌握其使用场景和优势。

---

## Q2：什么是 Stream Gatherers？它解决了什么问题？

**A：**
Stream Gatherers 是 JDK 24 正式引入的特性，允许开发者定义**自定义的 Stream 中间操作**，填补了 JDK 8~21 只能使用固定中间操作的空白。

**核心概念：**
Gatherer 有四个核心方法：
```java
public interface Gatherer<T, R> {
    Supplier<R> initializer();                    // 初始化状态
    Gatherer.Integrator<A, T, R> integrator();     // 处理每个元素
    BinaryOperator<A> combiner();                  // 并行合并
    BiConsumer<A, Downstream<? super R>> finisher(); // 完成时处理
}
```

**使用场景示例：**
```java
// 固定窗口分组（JDK 8~23 需要手动实现）
Gatherer<Integer, ?, List<Integer>> fixedWindow(int size) {
    return Gatherer.ofSequential(
        ArrayList::new,
        (list, item, downstream) -> {
            list.add(item);
            if (list.size() == size) {
                downstream.push(new ArrayList<>(list));
                list.clear();
            }
            return true;
        },
        (list, downstream) -> {
            if (!list.isEmpty()) downstream.push(list);
        }
    );
}

// 使用
List<List<Integer>> windows = IntStream.range(1, 10)
    .boxed()
    .gather(fixedWindow(3))
    .toList();
// [[1,2,3], [4,5,6], [7,8,9]]
```

**解决了什么问题：**
- 窗口滑动、去重、自定义聚合等操作不再需要 Collector
- 代码更直观，复用性更强
- 内置 `Gatherer.ofGreedySet()` 可直接去重

---

## Q3：Foreign Function & Memory API（FFM API）相比 JNI 有哪些优势？

**A：**
FFM API（JDK 24 正式）是 Java 调用本地原生代码的新标准，完全替代 JNI：

| 对比项 | JNI | FFM API |
| --- | --- | --- |
| **代码复杂度** | 需要 C 头文件、native 方法声明 | 纯 Java 代码 |
| **内存管理** | 手动管理 ByteBuffer | 自动 Arena 内存管理 |
| **类型安全** | 低（直接内存地址操作） | 高（MemorySegment 抽象） |
| **性能** | 较高但有 JNI 调用开销 | 优化后性能更好 |
| **依赖** | 需要 native 编译器 | 无需额外依赖 |

**基本用法：**
```java
import java.lang.foreign.*;

// 1. 加载本地库
MemorySegment libc = SymbolLookup.libraryLookup("c", Arena.global());

// 2. 定义函数签名
FunctionDescriptor strlenType = FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS);

// 3. 获取方法句柄
MethodHandle strlen = Linker.nativeLinker().downcallHandle(
    libc.find("strlen").orElseThrow(), strlenType
);

// 4. 调用
try (Arena arena = Arena.ofConfined()) {
    MemorySegment cString = arena.allocateUtf8String("Hello");
    long len = (long) strlen.invoke(cString);  // 输出 5
}
```

**面试重点**：FFM API 是 Java 21~24 最重磅的特性之一，面试中问到 FFM 时，能说出"纯 Java 调用 native 代码、Arena 自动内存管理"即可。

---

## Q4：Scoped Values 和 ThreadLocal 有什么区别？何时使用 Scoped Values？

**A：**
Scoped Values（JDK 24 正式）是比 ThreadLocal 更安全的跨线程数据共享机制：

| 对比项 | ThreadLocal | Scoped Values |
| --- | --- | --- |
| **可变性** | 可变（set/get） | **不可变**（创建后不可修改） |
| **跨线程传递** | 需手动传递 | **自动继承到子虚拟线程** |
| **内存效率** | 随线程数线性增长 | 更高效，子线程共享父线程的值 |
| **安全性** | 容易忘记 remove() 导致内存泄漏 | 生命周期清晰 |
| **适用场景** | 普通多线程 | **虚拟线程 + structured concurrency** |

**ThreadLocal 的问题：**
```java
ThreadLocal<User> userContext = new ThreadLocal<>();
userContext.set(currentUser);
// 忘记调用 userContext.remove() 会导致内存泄漏
```

**Scoped Values 的优势：**
```java
static final ScopedValue<String> USER = ScopedValue.newInstance();

// 父线程设置，子虚拟线程自动可见
ScopedValue.where(USER, "Alice", () -> {
    CompletableFuture.runAsync(() -> {
        System.out.println(USER.get());  // ✅ 输出 "Alice"，无需传递
    });
    // Structured Concurrency 中自动继承
});
```

**何时使用：**
- 虚拟线程 + 结构化并发 → 使用 Scoped Values
- 传统线程池 → ThreadLocal 仍是首选

---

## Q5：什么是压缩字符串（JDK 24/25）？为什么 Latin-1 字符串更省内存？

**A：**
JDK 24 正式引入压缩字符串，对 Latin-1 字符使用紧凑编码：

**原理：**
```java
// 之前（JDK 8~23）：
// 每个 String 内部 char[] 存储，每个字符固定 2 字节
String s1 = "Hello";  // char[5] = 10 bytes

// 之后（JDK 24+）：
// Latin-1 字符用 byte[] 存储，每个字符 1 字节
String s1 = "Hello";  // byte[5] = 5 bytes（节省 50%）
String s2 = "你好";    // UTF-16 仍用 char[]，2 chars = 4 bytes
```

**编码选择规则：**
- 字符串只包含 Latin-1 字符（0x00~0xFF）→ 使用 `byte[]`，1 字节/字符
- 字符串包含其他 Unicode 字符 → 使用 `char[]`，2 字节/字符
- JDK 24/25 会自动选择最优编码，对上层 API 完全透明

**内存对比：**
| 字符串内容 | JDK 8~23 | JDK 24/25 | 节省 |
| --- | --- | --- | --- |
| `"Hello World"` | 22 bytes (含对象头) | 22 bytes | 50%（内容区） |
| `"你好"` | 22 bytes | 22 bytes | 无变化 |

**面试重点**：压缩字符串是 JDK 24 的重要特性，面试中可以简述"Latin-1 用 byte，Unicode 用 char"的规则。

---

## Q6：什么是 String Templates（JDK 21~23 预览版）？它和 String.format() 有什么区别？

**A：**
String Templates 是 JDK 21~23 预览的特性，提供更直观的字符串插值语法：

**基本语法：**
```java
String name = "Alice";
int age = 30;

// 模板处理器 STR：直接插值
String msg = STR."Hello \{name}, you are \{age} years old.";

// 等价于 String.format()
String msg2 = String.format("Hello %s, you are %d years old.", name, age);
```

**FMT 模板处理器（格式化控制）：**
```java
String formatted = FMT."""
    Name:  \{name,-10}   // 左对齐，占10位
    Age:   \{age,5}      // 右对齐，占5位
    Price: \{123.456,10.2f}  // 浮点格式化
    """;
```

**多行模板（Text Blocks 的增强）：**
```java
String html = STR."""
    <div>
        <h1>\{title}</h1>
        <p>\{content}</p>
    </div>
    """;
```

**自定义模板处理器：**
```java
// RAW 获取原始模板，PROCESS 应用自定义处理器
String custom = RAW."Result: \{42}".process(myProcessor);
```

**注意**：String Templates 在 JDK 23 后暂时搁置，未进入 JDK 24/25 正式版，学习时了解即可。

---

## Q7：什么是 Unnamed Patterns and Variables（unnamed ~）？使用场景是什么？

**A：**
Unnamed Patterns and Variables（JDK 22 正式）是 JDK 21 预览后正式化的特性，用 `_` 表示"忽略不需要的值"：

**语法示例：**
```java
// 1. Unnamed Pattern：忽略嵌套数据结构
switch (shape) {
    case Circle(var radius)         -> process(radius);
    case Rectangle(int width, int _) -> process(width);  // 忽略 height
    case Triangle                    -> processDefault();
}

// 2. Unnamed Variable：忽略未使用的变量
map.forEach((_, value) -> process(value));  // 只要 value

// 3. Unnamed Exception：忽略异常变量
try {
    doSomething();
} catch (IOException _) {  // 只要错误类型
    log.error("Failed");
}

// 4. try-with-resources 中忽略资源变量
try (var _ = ScopedValue.get()) {
    // 只关心 ScopedValue 效果
}
```

**解决了什么问题：**
- 避免引入无意义的变量名（如 `unused`）
- 编译器对 `_` 的使用更友好
- Lambda 参数未使用时更简洁：`list.filter(_ -> true)`

**面试重点**：这个特性比较小众，但能体现对 JDK 新特性的了解，面试中能举出 1~2 个使用场景即可。

---

## Q8：JDK 22 的 Statements before super() 解决了什么问题？

**A：**
JDK 22 允许在 `super()` 调用之前执行语句，解决了之前的限制：

**JDK 21 及之前的限制：**
```java
class Parent {
    Parent(int x) { /* ... */ }
}

class Child extends Parent {
    Child() {
        // ❌ 编译错误：super() 必须在构造方法第一行
        int localVar = validate();  
        super(localVar);
    }
}
```

**JDK 22 的改进：**
```java
class Child extends Parent {
    Child() {
        int localVar = validate();      // ✅ JDK 22 允许
        super(localVar);                // ✅ super() 不必是第一行
        postInit();                     // ✅ 也可以在 super() 之后
    }
}
```

**使用场景：**
- 构造前参数校验和转换
- 计算父类构造方法需要的参数
- 初始化与父类无关的字段后再调用父类构造

**注意事项：**
- `this()` 调用仍必须在 `super()` 之前
- `super()` 仍必须最终被调用

**面试重点**：这是一个相对较小的语法改进，知道有这么回事即可。

---

## Q9：JDK 25 中 Primitive Patterns in instanceof 是什么？

**A：**
JDK 25 正式引入了 `instanceof` 对原始类型的模式匹配：

**JDK 24 及之前的写法：**
```java
void process(Object obj) {
    if (obj instanceof Integer) {
        Integer i = (Integer) obj;  // 需要强制转换
        System.out.println("Integer: " + i);
    } else if (obj instanceof Double) {
        Double d = (Double) obj;    // 需要强制转换
        System.out.println("Double: " + d);
    }
}
```

**JDK 25 的改进：**
```java
void process(Object obj) {
    // ✅ instanceof 直接绑定到局部变量，无需强制转换
    if (obj instanceof int i) {           // int（非 Integer）
        System.out.println("Integer: " + i);
    } else if (obj instanceof double d) { // double（非 Double）
        System.out.println("Double: " + d);
    }
}
```

**支持的原始类型：**
- `byte`、`short`、`char`
- `int`、`long`、`float`、`double`
- `boolean`

**注意**：绑定的是**包装类型对应的原始类型**值，不是装箱后的对象。

---

## Q10：Import Declarations（JDK 21~25 预览/正式）是什么？

**A：**
JDK 25 正式引入 Import Declarations（导入声明），允许批量导入一个包下的所有公开类型：

**语法：**
```java
// 无名导入：导入包下所有公开类型
import package com.example.utils.*;

// 使用时无需加类名前缀
var list = new ArrayList<>();     // ✅ ArrayList
var map = new HashMap<>();         // ✅ HashMap
var set = new HashSet<>();         // ✅ HashSet
```

**对比传统导入：**
```java
// 传统方式：需要显式导入每个类
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

// Import Declarations：批量导入
import package java.util.*;
```

**注意事项：**
- 不会导入 `package-info` 或 `module-info`
- 如果两个包有同名类，编译器会报错（和传统导入一致）
- 建议仅在工具类或测试代码中使用，避免命名冲突

**面试重点**：Import Declarations 是 JDK 25 的新特性，目前使用较少，了解即可。
