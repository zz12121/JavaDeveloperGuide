---
title: this关键字
tags:
  - Java/面向对象
  - 原理型
module: 02_面向对象
created: 2026-04-18
---

# this 关键字

## 先说结论

`this` 指向**当前对象实例**，是一个隐式引用。主要用于区分实例变量和局部变量、调用本类的其他构造方法、将当前对象作为参数传递。

## 深度解析

### this 的三种用法

| 用法 | 语法 | 说明 |
|------|------|------|
| 引用当前对象属性 | `this.name` | 区分实例变量和局部变量 |
| 调用本类构造方法 | `this(args)` | 必须在构造方法第一行 |
| 作为参数传递 | `method(this)` | 将当前对象传递给其他方法 |
| 作为返回值 | `return this` | 支持链式调用 |

### this 的本质
- `this` 是一个**引用变量**，存储在栈中的局部变量表里
- 指向堆中当前正在被操作的对象
- 只能在**实例方法**和**构造方法**中使用，static 方法中不能用 this

### this 和 super 不能共存
- `this()` 和 `super()` 都必须在构造方法第一行
- 因此一个构造方法中不能同时出现 `this()` 和 `super()`

## 易错点/踩坑

- ❌ "static 方法中可以用 this"：static 方法属于类，没有当前对象实例，不能用 this
- ❌ "this 指向类本身"：this 指向的是**对象实例**，不是类
- ❌ "内部类的 this 和外部类一样"：内部类的 `this` 指向内部类实例，访问外部类用 `Outer.this`

## 代码示例

```java
public class Person {
    private String name;
    private int age;

    // this 区分属性和参数
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // this() 调用本类其他构造
    public Person(String name) {
        this(name, 0);  // 必须在第一行
    }

    // this 作为参数传递
    public void register() {
        EventManager.register(this);  // 传入当前对象
    }

    // this 作为返回值（链式调用）
    public Person setName(String name) {
        this.name = name;
        return this;  // 返回当前对象
    }

    // 链式调用
    // new Person().setName("张三").setAge(25);
}
```

## 图解/流程

```
Person p = new Person("张三");

栈                    堆
───                  ────
p ──────→ ┌──────────────────┐
          │ Person 对象      │
this ─────│ name = "张三"    │  ← this 指向当前对象
          │ age  = 25        │
          └──────────────────┘

在 setName("李四") 方法中：
this.name = name;
 ↑            ↑
 │            └── 参数（局部变量）
 └── 实例变量（对象属性）
```

---

## 关联知识点
