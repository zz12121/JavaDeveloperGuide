---
id: qa_2
title: main 方法详解
tags:
  - Java/基础语法
  - 原理型
  - 问答
module: 01_基础语法
created: 2026-04-18
---

# main 方法详解（public static void main(String[] args)）

## Q1：每个关键字的作用？

**A**：


| 关键字 | 作用 | 去掉会怎样 |
| :---: | :---: | :--- |
| `public` | JVM 需从外部调用 main，必须公开 | 运行时找不到 main 方法 |
| `static` | JVM 无需创建对象即可直接调用 | 必须先 new 对象才能运行，但谁来 new？ |
| `void` | main 是程序起点，返回值无意义 | JVM 无法处理返回值 |
| `main` | 约定方法名，JVM 识别入口 | JVM 找不到入口方法 |
| `String[] args` | 接收命令行参数 | 签名不匹配，JVM 不认 |

---

## Q2：args 的使用？

**A**：
```java
// 命令行：java MyApp hello world
public static void main(String[] args) {
    System.out.println(args[0]); // hello
    System.out.println(args[1]); // world
}
```

- `args` 可以为空数组（不传参时），但不会是 null
- JDK 5+ 也支持 `String... args`（可变参数），但本质一样

---

## Q3：main 方法的变体与限制？

**A**： - **不能重写** main 方法（static 方法隐藏，不是多态）
- **可以重载** main 方法，但 JVM 只认 `public static void main(String[] args)` 这个签名
- JDK 11+ 支持**单文件源码直接运行**：`java Hello.java`（省略 javac）

---

## Q4：为什么 main 方法必须用 static 修饰？

**A**：JVM 启动时还没有任何对象存在，无法通过 `new` 创建对象再调用方法。static 方法属于类，可以直接通过类名调用，所以 JVM 可以直接调用 `ClassName.main()`。

---

## Q5：main 方法可以重写吗？可以重载吗？

**A**：不能重写（override），因为 static 方法不存在多态，子类定义同名方法叫"隐藏"。可以重载，但 JVM 只认标准签名的 main 方法作为入口。

---

## Q6：一个类可以有多个 main 方法吗？

**A**：可以重载多个 main 方法（如 `main(int[])`），但只有 `public static void main(String[] args)` 会被 JVM 识别为程序入口。

---

## 关联知识点

