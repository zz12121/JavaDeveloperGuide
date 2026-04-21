---
title: static 关键字
tags:
  - Java/面向对象
  - 原理型
  - 问答
module: 02_面向对象
created: 2026-04-21
---

# static 关键字

## 先说结论

`static` 表示**属于类本身，而不属于某个对象**。被 static 修饰的成员（属性、方法、代码块、内部类）在类加载时就存在，所有对象共享，通过类名直接访问。

---

## 深度解析

### 四种 static 用法总览

| 用法 | 语法形式 | 核心特点 |
|------|---------|---------|
| 静态属性 | `static int count;` | 所有对象共享同一份，类级别变量 |
| 静态方法 | `static void method()` | 无需实例调用，不能访问实例成员 |
| 静态代码块 | `static { ... }` | 类加载时执行一次，用于静态初始化 |
| 静态内部类 | `static class Inner {}` | 不依赖外部类实例，可独立创建 |

---

### 静态属性（Static Field）

```java
public class Counter {
    public static int count = 0;  // 类变量，所有实例共享

    public Counter() {
        count++;  // 每创建一个对象，count + 1
    }
}

// 使用
Counter c1 = new Counter();
Counter c2 = new Counter();
System.out.println(Counter.count);  // 2（推荐：类名.静态属性）
```

**内存位置**：静态属性存储在**方法区（元空间）**中，不在堆上的对象实例里。  
**生命周期**：随类的加载而创建，随类的卸载而销毁，比对象生命周期更长。

---

### 静态方法（Static Method）

```java
public class MathUtil {
    public static int add(int a, int b) {
        return a + b;
    }
}

// 使用（无需实例化）
int result = MathUtil.add(3, 5);
```

**核心限制**：
- 静态方法中**不能使用 `this`、`super`**（没有当前对象上下文）
- 静态方法**不能直接访问实例变量和实例方法**（因为此时可能没有对象）
- 实例方法可以调用静态方法；反之静态方法不能直接调用实例方法

**常见场景**：工具类方法（`Math.abs()`、`Collections.sort()`）、工厂方法、`main` 方法。

---

### 静态代码块（Static Initializer Block）

```java
public class Config {
    public static final Map<String, String> CONFIG_MAP;

    static {
        // 类加载时执行，且只执行一次
        CONFIG_MAP = new HashMap<>();
        CONFIG_MAP.put("host", "localhost");
        CONFIG_MAP.put("port", "8080");
        System.out.println("Config loaded.");
    }
}
```

**执行时机与顺序**：

```
类加载阶段执行顺序：
1. 静态属性（按代码顺序初始化）
2. 静态代码块（按代码顺序执行）
↓
实例化阶段执行顺序：
3. 实例属性
4. 实例代码块（普通代码块 {}）
5. 构造方法
```

**关键特性**：
- 只在类第一次加载时执行**一次**，之后不再执行
- 多个静态代码块按**从上到下**的顺序执行
- 常用于：加载驱动、初始化常量集合、注册工厂

---

### 静态内部类（Static Nested Class）

```java
public class Outer {
    private static int outerStatic = 10;
    private int outerInstance = 20;

    // 静态内部类
    public static class StaticNested {
        public void show() {
            System.out.println(outerStatic);     // ✅ 可以访问外部类静态成员
            // System.out.println(outerInstance); // ❌ 不能访问外部类实例成员
        }
    }
}

// 创建静态内部类实例（不需要外部类实例）
Outer.StaticNested nested = new Outer.StaticNested();
nested.show();
```

**与普通内部类对比**：

| 特性 | 普通内部类 | 静态内部类 |
|------|-----------|-----------|
| 创建方式 | 需要外部类实例 `outer.new Inner()` | 直接 `new Outer.Inner()` |
| 持有外部类引用 | 隐式持有（可能内存泄漏） | 不持有 |
| 访问外部类成员 | 静态 + 实例成员都可访问 | 只能访问静态成员 |
| 典型用途 | Builder、Iterator | Builder（单独场景）、枚举辅助 |

**经典应用**：`Builder` 模式中的 Builder 类、`LinkedList.Node`、`HashMap.Entry` 都是静态内部类。

---

## 易错点/踩坑

- ❌ "静态方法中可以用 this/super"：不能，static 方法没有对象上下文
- ❌ "静态属性存在堆里"：存在方法区（元空间），不在堆中的对象实例里
- ❌ "静态代码块每次创建对象都执行"：只在类加载时执行一次
- ❌ "可以用实例访问静态属性/方法"：语法允许，但强烈不推荐（IDE 会警告），应用类名访问
- ❌ "静态内部类能访问外部类所有成员"：只能访问外部类的 **static** 成员
- ❌ "final static 和 static final 有区别"：没有区别，修饰符顺序不影响语义，但习惯写 `static final`

---

## 代码示例（综合）

```java
public class Singleton {
    // 静态属性：持有唯一实例
    private static Singleton instance;

    // 静态代码块：类加载时初始化
    static {
        instance = new Singleton();
        System.out.println("Singleton initialized.");
    }

    // 私有构造（不允许外部 new）
    private Singleton() {}

    // 静态方法：对外提供唯一入口
    public static Singleton getInstance() {
        return instance;
    }

    // 静态内部类版本（推荐的懒加载单例）
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstanceLazy() {
        return Holder.INSTANCE;  // 类加载时才初始化，线程安全
    }
}
```

---

## 图解/流程

```
JVM 内存中 static 成员的位置：

┌──────────────────────────────────────┐
│         方法区（元空间 Metaspace）       │
│  ┌──────────────────────────────┐    │
│  │ Counter.class                 │    │
│  │  ├── static int count = 2    │ ← 静态属性在这里
│  │  ├── static { ... } 已执行   │ ← 静态代码块执行结果
│  │  └── static add() 方法引用   │ ← 静态方法在这里
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│              堆（Heap）               │
│  ┌─────────┐   ┌─────────┐          │
│  │ obj1    │   │ obj2    │          │
│  │ (实例)  │   │ (实例)  │          │
│  └─────────┘   └─────────┘          │
│  ← 对象实例在这里，没有 static 成员 →  │
└──────────────────────────────────────┘
```

---

## 关联知识点

- [[super关键字]] - static 方法中不能用 super
- [[this关键字]] - static 方法中不能用 this
- [[单例模式]] - 静态内部类实现线程安全懒加载单例
- [[类加载机制]] - static 块的执行时机与类初始化阶段
- [[内部类]] - 静态内部类 vs 普通内部类区别
