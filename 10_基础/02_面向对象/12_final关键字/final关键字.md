---
title: final 关键字
tags:
  - Java/面向对象
  - 原理型
  - 问答
module: 02_面向对象
created: 2026-04-21
---

# final 关键字

## 先说结论

`final` 表示**不可改变**。修饰变量时值不可重新赋值，修饰方法时不可被重写，修饰类时不可被继承。它是 Java 中表达"封闭性"和"不变性"的核心关键字。

---

## 深度解析

### 三种 final 用法总览

| 用法 | 语法形式 | 核心含义 |
|------|---------|---------|
| final 变量 | `final int x = 10;` | 只能赋值一次，之后不可修改 |
| final 方法 | `final void method()` | 子类不能重写（Override）此方法 |
| final 类 | `final class MyClass {}` | 此类不能被继承 |

---

### final 变量（Final Variable）

#### 1. final 局部变量
```java
final int x = 10;
x = 20;  // ❌ 编译错误：无法为最终变量 x 分配值
```
声明时可以不立即赋值（空白 final），但**使用前必须赋值且只能赋值一次**：
```java
final int y;  // 空白 final
y = 100;      // ✅ 第一次赋值
y = 200;      // ❌ 编译错误
```

#### 2. final 实例变量
必须在以下三个时机之一完成赋值，否则编译报错：
```java
public class Person {
    // 方式一：声明时直接赋值
    final String type = "human";

    // 方式二：实例代码块中赋值
    final int id;
    { id = nextId(); }

    // 方式三：构造方法中赋值（每个构造方法都要保证赋值）
    final String name;
    public Person(String name) {
        this.name = name;
    }
}
```

#### 3. final 静态变量（常量）
```java
public static final double PI = 3.14159265358979;
public static final int MAX_RETRY = 3;
```
必须在**声明时**或**静态代码块**中赋值：
```java
public static final Map<String, Integer> CODE_MAP;
static {
    CODE_MAP = new HashMap<>();
    CODE_MAP.put("OK", 200);
}
```

#### 4. ⚠️ final 引用类型的陷阱
```java
final List<String> list = new ArrayList<>();
list.add("hello");   // ✅ 可以修改对象内容
list.add("world");   // ✅ 没问题
list = new ArrayList<>();  // ❌ 不能重新指向新对象
```
**final 保证的是引用本身不变（不能重新指向），而不是引用所指向的对象内容不变。**

---

### final 方法（Final Method）

```java
public class Base {
    public final void template() {
        // 模板方法，子类不得修改
        step1();
        step2();
    }

    protected void step1() { System.out.println("Base step1"); }
    protected void step2() { System.out.println("Base step2"); }
}

public class Sub extends Base {
    @Override
    public void template() { }  // ❌ 编译错误：不能重写 final 方法

    @Override
    protected void step1() { System.out.println("Sub step1"); }  // ✅ 可以重写非 final 方法
}
```

**使用场景**：
- **模板方法模式**：固定算法骨架，禁止子类破坏流程
- **安全方法**：涉及安全验证、核心业务逻辑，不允许子类篡改行为

**注意**：`private` 方法隐式就是 final（子类根本看不到，无法重写），加不加 final 关键字效果相同。

---

### final 类（Final Class）

```java
public final class ImmutablePoint {
    private final int x;
    private final int y;

    public ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }
}

public class Point3D extends ImmutablePoint { }  // ❌ 编译错误：无法从最终 ImmutablePoint 继承
```

**JDK 中的 final 类**：
- `String`：字符串不可变，保证字符串常量池的安全和哈希一致性
- `Integer`、`Long` 等包装类：不可变对象，线程安全
- `Math`：纯工具类，无需继承扩展

**final 类的特点**：
- 类中所有方法**隐式为 final**（既然不能继承，重写也无从谈起）
- 是实现**不可变对象（Immutable Object）**的重要手段之一

---

### final 与性能优化

- **编译期常量内联**：`static final` 的基本类型或 String 常量，编译器会直接将值内联到调用处，无运行时访问开销
- **JIT 优化**：JVM 对 final 方法可以做内联（inline）优化，避免动态分派（虚方法查找）的开销
- **内存可见性**：`final` 字段有特殊的内存语义——在构造方法结束后，所有线程都能看到 final 字段的正确初始化值（无需同步），这是 `String` 线程安全的基础之一

---

## 易错点/踩坑

- ❌ "final 引用变量，对象内容也不可变"：final 只保证引用不变，对象内容仍可修改（除非对象本身是不可变的）
- ❌ "final 方法不能被继承"：可以继承，只是不能被**重写**
- ❌ "final 类的成员自动变 final"：类中方法隐式为 final，但实例变量不会自动变 final
- ❌ "private 方法和 final 方法效果不同"：private 方法子类本就看不到，隐式就是 final，加 final 无意义
- ❌ "final 变量必须声明时赋值"：实例 final 变量可以在构造方法中赋值（空白 final）
- ❌ "static final 和 final static 有区别"：没有区别，修饰符顺序不影响语义

---

## 代码示例（综合——不可变类）

```java
/**
 * 标准不可变类设计：
 * 1. 类用 final 修饰（防止子类破坏不变性）
 * 2. 所有字段用 private final 修饰
 * 3. 不提供 setter
 * 4. 构造方法中完成所有初始化
 * 5. 若字段是可变对象（如数组/集合），构造时做防御性拷贝
 */
public final class ImmutableUser {
    private final String name;
    private final int age;
    private final List<String> roles;

    public ImmutableUser(String name, int age, List<String> roles) {
        this.name = name;
        this.age = age;
        // 防御性拷贝：防止外部修改传入的 list
        this.roles = Collections.unmodifiableList(new ArrayList<>(roles));
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public List<String> getRoles() { return roles; }  // 返回不可修改视图
}
```

---

## 图解/流程

```
final 的三种封闭层次：

变量级别（值不变）：
  final int x = 10; ──► x 的值永远是 10，不可重新赋值

方法级别（行为不变）：
  ┌──────────┐              ┌──────────┐
  │  Base    │              │   Sub    │
  │ final    │  继承  →     │  ✅可继承 │
  │ method() │              │  ❌不可重写│
  └──────────┘              └──────────┘

类级别（结构不变）：
  final class String
       ↑
       ❌ 任何类都不能继承 String

引用类型的 final（注意区分！）：
  final List list ──► list 引用不变 ──► list 内容仍可 add/remove
                              ↕
              list = new List() ❌（引用不可改）
```

---

## 关联知识点

- [[static关键字]] - `static final` 组合定义编译期常量
- [[不可变对象]] - final 是实现不可变类的核心手段
- [[String类]] - String 是 final 类，理解其不可变性
- [[多态]] - final 方法阻止方法重写，破坏多态
- [[线程安全]] - final 字段的内存语义保证可见性
