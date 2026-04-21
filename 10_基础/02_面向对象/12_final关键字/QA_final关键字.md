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

## Q1：final 的三种用法分别是什么？

**A**：
1. **final 变量**：只能赋值一次，之后不可修改（基本类型值不变，引用类型引用不变）
2. **final 方法**：子类可以继承，但不能重写（Override）
3. **final 类**：该类不能被继承，类中所有方法隐式为 final

---

## Q2：final 实例变量必须在哪里赋值？

**A**：final 实例变量必须在以下三处之一完成赋值，否则编译报错：
1. **声明时直接赋值**：`final int id = 1;`
2. **实例代码块中**：`{ id = nextId(); }`
3. **构造方法中**：每个构造方法都必须保证 final 变量被赋值（若有多个构造方法，每个都要赋值）

不能在普通方法中赋值，也不能在静态代码块中赋值（静态代码块只能初始化静态变量）。

---

## Q3：final 引用类型变量，对象内容能修改吗？

**A**：**可以**。`final` 只保证引用本身不能重新指向新对象，但对象的内部状态（内容）仍然可以修改。

```java
final List<String> list = new ArrayList<>();
list.add("hello");        // ✅ 可以修改内容
list = new ArrayList<>(); // ❌ 不能重新赋值引用
```

要让对象内容也不可变，需要额外措施（如 `Collections.unmodifiableList()`、使用不可变类等）。

---

## Q4：final 方法和 private 方法有什么区别？

**A**：

| 对比维度 | final 方法 | private 方法 |
|---------|-----------|-------------|
| 可见性 | 子类可以继承和调用 | 子类根本看不到 |
| 重写 | 不能重写（编译报错） | 无法重写（子类定义同名方法是新方法，非重写） |
| 多态 | 不参与多态（不走虚方法表） | 不参与多态 |
| 关系 | `private` 方法隐式就是 `final` | — |

**结论**：`private` 方法自动具有 `final` 语义，在 `private` 方法上显式加 `final` 关键字是多余的。

---

## Q5：为什么 String 类要设计成 final？

**A**：String 设计为 final 类主要出于以下几点考虑：

1. **字符串常量池安全**：String 不可变才能保证同一字面量在常量池中只有一份，多处引用同一对象安全
2. **哈希值缓存**：`String.hashCode()` 结果被缓存，不可变保证哈希值永远一致
3. **线程安全**：不可变对象天然线程安全，String 可以在多线程间自由共享
4. **防止继承破坏**：若允许继承，子类可以重写方法改变行为，破坏上述所有保证
5. **安全性**：用于类加载、网络连接、文件路径等场景，不可变防止被恶意篡改

---

## Q6：如何设计一个标准的不可变类？

**A**：设计不可变类需要满足以下条件：

```java
public final class ImmutableUser {           // 1. 类用 final 修饰
    private final String name;               // 2. 所有字段 private final
    private final List<String> roles;

    public ImmutableUser(String name, List<String> roles) {
        this.name = name;
        // 3. 对可变引用类型做防御性拷贝
        this.roles = Collections.unmodifiableList(new ArrayList<>(roles));
    }

    public String getName() { return name; }
    // 4. 不提供任何 setter
    // 5. 若返回可变对象，也要做防御性拷贝或返回不可修改视图
    public List<String> getRoles() { return roles; }
}
```

**五要素**：
1. 类加 `final`（防止子类破坏）
2. 所有字段 `private final`
3. 不提供 setter
4. 构造中完成所有初始化
5. 可变对象字段做防御性拷贝

---

## Q7：final 变量会被编译器内联优化吗？

**A**：**会，但有条件**。  
当 `static final` 变量的值是**编译期常量**（基本类型或 `String` 字面量）时，编译器会将其内联（常量折叠）到所有引用处：

```java
public static final int MAX = 100;
// 使用时
if (size > MAX) { ... }
// 编译后等价于：
if (size > 100) { ... }   // 直接用值，无访问开销
```

**不会内联的情况**：
- `final` 实例变量（非 static）
- 值是运行时计算的：`static final int X = new Random().nextInt();`
- 值是对象引用（非基本类型/String）

---

## Q8：final 和线程安全有什么关系？

**A**：`final` 字段有特殊的 **JMM（Java内存模型）语义**：

> 只要对象的构造方法正确完成（未发生引用逃逸），所有线程读取该对象的 `final` 字段时，都能看到构造方法中写入的正确值，**无需额外同步**。

这就是 `String`、不可变包装类等在多线程中安全共享的底层保障。  
**注意**：这个保证只针对 final 字段本身，若 final 字段指向的是可变对象，对象内部的修改仍需同步。

---

## Q9：final、finally、finalize 三者的区别？

**A**：三个完全不同的概念，只是拼写相似：

| 关键字 | 类型 | 含义 |
|--------|------|------|
| `final` | 修饰符关键字 | 修饰变量/方法/类，表示不可变/不可重写/不可继承 |
| `finally` | 异常处理关键字 | `try-catch-finally` 中，无论是否抛出异常都会执行的代码块 |
| `finalize` | Object 方法 | GC 回收对象前调用的方法（Java 9 已废弃，不可靠） |

---

## Q10：子类能重写父类的 final 方法吗？能隐藏吗？

**A**：
- **重写（Override）**：❌ 不能，编译直接报错
- **隐藏（Hide）**：❌ 也不能，final 方法同样不能被子类以 `static` 方式隐藏

但要注意：子类可以**继承**并**调用** final 方法（它仍对子类可见），只是不能改变其行为。

```java
class Base {
    public final void show() { System.out.println("Base"); }
}

class Sub extends Base {
    // @Override 不能重写
    // public void show() { }  // ❌ 编译错误

    public void test() {
        show();        // ✅ 可以继承并调用
        super.show();  // ✅ 也可以这样调用
    }
}
```

---

## 关联知识点

- [[static关键字]] - `static final` 组合定义编译期常量
- [[不可变对象]] - final 是实现不可变类的核心手段
- [[String类]] - String 是 final 类，理解其不可变性
- [[多态]] - final 方法阻止方法重写，破坏多态
- [[线程安全]] - final 字段的内存语义保证可见性
