---
title: main 方法详解
tags:
  - Java/基础语法
  - 原理型
module: 01_基础语法
created: 2026-04-18
---

# main 方法详解（public static void main(String[] args)）

## 先说结论

`public static void main(String[] args)` 是 Java 程序的入口方法，每个修饰符都有其存在的必要原因：`public` 供 JVM 外部调用、`static` 无需实例化即可运行、`void` 无返回值、`String[]` 接收命令行参数。

## 深度解析

### 每个关键字的作用

| 关键字 | 作用 | 去掉会怎样 |
|--------|------|-----------|
| `public` | JVM 需从外部调用 main，必须公开 | 运行时找不到 main 方法 |
| `static` | JVM 无需创建对象即可直接调用 | 必须先 new 对象才能运行，但谁来 new？ |
| `void` | main 是程序起点，返回值无意义 | JVM 无法处理返回值 |
| `main` | 约定方法名，JVM 识别入口 | JVM 找不到入口方法 |
| `String[] args` | 接收命令行参数 | 签名不匹配，JVM 不认 |

### args 的使用
- `args` 可以为空数组（不传参时），但不会是 null
- JDK 5+ 也支持 `String... args`（可变参数），但本质一样

### main 方法的变体与限制
- **不能重写** main 方法（static 方法隐藏，不是多态）
- **可以重载** main 方法，但 JVM 只认 `public static void main(String[] args)` 这个签名
- JDK 11+ 支持**单文件源码直接运行**：`java Hello.java`（省略 javac）

## 易错点/踩坑

- ❌ main 方法不能被重写（override）：static 方法不存在多态，子类定义同名 static 方法叫"隐藏"（hide），不是重写
- ❌ `String... args` 和 `String[] args` 对 JVM 来说是一样的签名，但只有 `String[]` 写法是官方标准
- ❌ JVM 不认 `static void main()`（没有参数），签名必须完全匹配

## 代码示例

```java
// 命令行：java MyApp hello world
public static void main(String[] args) {
    System.out.println(args[0]); // hello
    System.out.println(args[1]); // world
    System.out.println(args.length); // 2
}
```

```java
// 重载 main — 编译通过，但 JVM 不认
public static void main(int[] args) {
    System.out.println("这个 main 不会被 JVM 调用");
}
```

## 图解/流程

```
┌───────────────────────────────────────────────────┐
│  public  static  void  main  (String[] args)      │
│    │       │      │      │         │               │
│    │       │      │      │         └─ 命令行参数    │
│    │       │      │      └─ 方法名(JVM约定)         │
│    │       │      └─ 无返回值(起点无接收者)          │
│    │       └─ 无需实例化(JVM直接调用)               │
│    └─ JVM外部访问需要公开                           │
└───────────────────────────────────────────────────┘
```

---

## 关联知识点

