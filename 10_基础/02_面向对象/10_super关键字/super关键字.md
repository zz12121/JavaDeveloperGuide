---
title: super 关键字
tags:
  - Java/面向对象
  - 原理型
  - 问答
module: 02_面向对象
created: 2026-04-18
---

# super 关键字

## 先说结论

`super` 指向**当前对象的父类部分**，用于访问父类被隐藏的成员变量、被重写的方法，以及调用父类构造方法。`this` 看自己，`super` 看父类。

## 深度解析

### super 的三种用法

| 用法 | 语法 | 说明 |
|------|------|------|
| 引用父类属性 | `super.name` | 访问被子类隐藏的父类属性 |
| 调用父类方法 | `super.method()` | 调用被子类重写的父类方法 |
| 调用父类构造 | `super(args)` | 必须在子类构造方法第一行 |

### super 的本质
- `super` 不是父类的引用，而是**当前对象中父类部分的标识**
- `super` 和 `this` 指向的是**同一个对象**，只是访问的视角不同
- `this.xxx` 先在子类找，找不到再去父类；`super.xxx` 直接去父类找

### super() 调用规则
- 子类构造方法**必须**调用父类构造方法（显式或隐式）
- 如果不显式写 `super()`，编译器自动插入 `super()`（调用父类无参构造）
- 如果父类没有无参构造，子类**必须**显式调用 `super(参数)`
- `super()` 必须在构造方法**第一行**

## 易错点/踩坑

- ❌ "super 是父类的引用"：super 不是引用变量，不能像 this 一样赋值或打印
- ❌ "super() 可以在任何地方调用"：只能在子类构造方法的第一行
- ❌ "super 可以访问父类的 private 成员"：不能，super 受访问修饰符限制
- ❌ "静态方法中可以用 super"：不能，和 this 一样，static 方法没有对象上下文

## 代码示例

```java
public class Animal {
    protected String name = "Animal";
    public void eat() { System.out.println("Animal eats"); }
    public Animal(String name) { this.name = name; }
}

public class Dog extends Animal {
    protected String name = "Dog";  // 隐藏父类属性

    public Dog(String name) {
        super(name);  // 调用父类构造（第一行）
    }

    @Override
    public void eat() {
        System.out.println("Dog eats bone");
        super.eat();  // 调用父类被重写的方法
    }

    public void test() {
        System.out.println(this.name);   // "Dog"（子类属性）
        System.out.println(super.name);  // "Animal"（父类属性，取决于构造传入）
    }
}
```

## 图解/流程

```
Dog 对象内存布局：

┌─────────────────────────┐
│ Animal 部分              │  ← super 访问这里
│ ├ name = "传入的name"    │
│ ├ eat() → Animal版本     │
├─────────────────────────┤
│ Dog 部分                 │  ← this 访问这里（优先）
│ ├ name = "Dog"           │
│ ├ eat() → Dog版本（重写）│
│ └ bark()                │
└─────────────────────────┘

this 和 super 指向同一个对象，只是"视角"不同
```

---

## 关联知识点
