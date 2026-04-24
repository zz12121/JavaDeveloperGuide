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


## 关联知识点

