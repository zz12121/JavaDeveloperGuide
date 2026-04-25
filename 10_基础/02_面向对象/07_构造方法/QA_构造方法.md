---
title: 构造方法面试题
tags:
  - Java/面向对象
  - 原理型
  - 问答
module: 02_面向对象
created: 2026-04-25
---

# 构造方法

## Q1：构造方法能被继承吗？能被子类调用吗？

**A：**
**构造方法不能被继承**。子类无法直接调用父类的构造方法，只能通过 `super()` 间接调用。

```java
class Parent {
    Parent() { System.out.println("Parent constructor"); }
}

class Child extends Parent {
    Child() {
        super();  // 必须显式或隐式调用父类构造方法
        System.out.println("Child constructor");
    }
}
```

---

## Q2：this() 和 super() 必须放在构造方法第一行的原因？

**A：**
因为构造方法是对象创建的**起点**，必须先初始化父类部分（或本类其他构造方法），再完成当前类的初始化：

```java
class Parent {
    Parent(int x) { }
}

class Child extends Parent {
    Child() {
        // ⚠️ 如果不写 super()，编译器自动插入 super()
        System.out.println("Child init");
    }
}
```

如果 this() 和 super() 不在第一行，父类/本类其他构造方法就无法在对象状态初始化之前执行，可能导致引用未初始化的字段。

---

## Q3：父类无参构造方法被覆盖后，子类不显式调用会怎样？

**A：**
如果父类只有有参构造方法，子类**必须显式调用** `super(param)`，否则编译错误：

```java
class Parent {
    Parent(int x) { }  // 没有无参构造方法！
}

class Child extends Parent {
    Child() {
        // ⚠️ 编译错误：隐式 super() 无法找到无参构造方法
    }

    Child() {
        super(0);  // ✅ 必须显式调用
    }
}
```

---

## Q4：static 方法中能用 this 和 super 吗？为什么？

**A：**
**不能**。static 方法属于类，在类加载时就存在，不依赖于任何对象实例。而 this 和 super 是实例级别的引用，必须依附于对象才能使用。

```java
class Example {
    static void staticMethod() {
        // this.toString();  // ⚠️ 编译错误
        // super.toString();  // ⚠️ 编译错误
    }

    void instanceMethod() {
        this.toString();  // ✅
        super.toString(); // ✅
    }
}
```

---

## Q5：简述子类实例化的构造方法调用顺序？

**A：**
1. **父类静态初始化块**（只执行一次）
2. **子类静态初始化块**（只执行一次）
3. **父类实例初始化块**
4. **父类构造方法**
5. **子类实例初始化块**
6. **子类构造方法**

```java
class Parent {
    static { System.out.println("1. Parent static"); }
    { System.out.println("3. Parent instance init"); }
    Parent() { System.out.println("4. Parent constructor"); }
}

class Child extends Parent {
    static { System.out.println("2. Child static"); }
    { System.out.println("5. Child instance init"); }
    Child() { System.out.println("6. Child constructor"); }
}

// new Child() 输出顺序：1 → 2 → 3 → 4 → 5 → 6
```

---

## Q6：构造方法中调用被重写的方法会发生什么？

**A：**
可能发生**构造函数继承问题（Constructor Inheritance Problem）**：

```java
class Parent {
    Parent() { print(); }  // 调用被子类重写的方法
    void print() { System.out.println("Parent"); }
}

class Child extends Parent {
    int x = 100;

    Child() { }
    @Override
    void print() {
        System.out.println(x);  // x 此时还是 0（未初始化）！
    }
}

new Child();  // 输出：0
```

子类实例字段还未初始化（x=0），但构造方法调用了被重写的 print()，导致读取到未初始化的值。

**原则：构造方法中尽量不要调用可被子类重写的方法。**

---

## Q7：this() 和 super() 能同时出现在同一个构造方法中吗？

**A：**
**不能**。`this()` 和 `super()` 都是构造方法调用的起点，只能有一个在第一行：

```java
class Parent {
    Parent(int x) { }
}

class Child extends Parent {
    Child() {
        // ⚠️ 编译错误：this() 和 super() 不能同时出现
        // super(0);
        // this(1);
    }

    Child(int x) {
        super(x);  // ✅ 第一行
    }
}
```

原因：`this(param)` 和 `super(param)` 都是调用另一个构造方法，被调用的构造方法必须先完成初始化工作。

---

## Q8：构造方法能返回值吗？

**A：**
**不能**。构造方法没有返回值类型（包括 void），编译器会自动在调用处插入对构造方法的调用：

```java
class User {
    User() { }  // 正确：没有返回值

    // User() { return; }  // ⚠️ 虽然语法上可以写 return，但不应该
}
```

如果构造方法需要返回对象，使用工厂方法模式（Factory Method）：

```java
class User {
    private User() { }

    // 工厂方法
    public static User create() {
        return new User();
    }
}
```
