---
title: 泛型接口面试题
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型接口

## Q1：如何定义和实现泛型接口？

**A：**
```java
// 定义
public interface Processor<T, R> {
    R process(T input);
}

// 实现方式1：指定具体类型
public class StringLengthProcessor implements Processor<String, Integer> {
    public Integer process(String input) { return input.length(); }
}

// 实现方式2：实现类保留类型参数
public class GenericProcessor<T, R> implements Processor<T, R> {
    public R process(T input) { ... }
}
```

---

## Q2：实现泛型接口时不指定类型参数会怎样？

**A：**
实现类会作为原始类型（Raw Type）处理，类型参数默认为 `Object`，并产生编译警告：

```java
// ⚠️ 警告：Raw type usage
public class RawProcessor implements Processor {
    public Object process(Object input) { ... }
}
```

---

## Q3：Comparable 和 Comparator 的区别？

**A：**

|      | `Comparable<T>`  | `Comparator<T>`       |
| ---- | ---------------- | --------------------- |
| 位置   | 对象自身实现           | 外部定义                  |
| 方法   | `compareTo(T o)` | `compare(T o1, T o2)` |
| 含义   | 自然排序             | 自定义排序                 |
| 修改原类 | 需要               | 不需要                   |
| 场景   | 一种固定顺序           | 多种排序策略                |
Comparable：我是谁（内在属性）
当一个类实现了 Comparable接口，意味着你定义了这个对象的“自然顺序”（Natural Ordering）。
语义：student1.compareTo(student2)翻译成人话就是：“告诉我，你和另一个学生相比，谁大谁小？”
侵入性：你需要修改 Student类的源代码。如果这个类是别人写的（比如 String、Integer），你改不了，那就没法用 Comparable。
单一性：一个类通常只有一种自然顺序。如果你强行写了第二种（比如既按年龄又按身高），代码会非常混乱。

Comparator：我想怎么排（外在策略）
Comparator是一个比较器，它是一个独立于对象之外的“裁判”。
语义：compare(s1, s2)翻译成人话就是：“裁判，你告诉我这两个学生谁大谁小？”
非侵入性：不需要动 Student类的代码。你可以随时定义无数种排序规则。
灵活性：这是它的杀手锏。你可以随时换裁判。

```java
// Comparable：修改类本身
class Student implements Comparable<Student> {
    public int compareTo(Student o) { return this.age - o.age; }
}

// Comparator：外部策略
Comparator<Student> byName = Comparator.comparing(Student::getName);
Comparator<Student> byAge = Comparator.comparingInt(Student::getAge);
list.sort(byName.thenComparing(byAge));

//thenComparing的威力（组合模式）
list.sort(byName.thenComparing(byAge));

```

为什么 `Comparator`能这么灵活地链式调用？
这得益于 **Java 8 引入的函数式接口和默认方法（Default Method）**。
查看 `Comparator`接口的源码（简化版）：
```java
@FunctionalInterface
public interface Comparator<T> {
    
    int compare(T o1, T o2);

    // 默认方法：接收一个 Function，返回一个新的 Comparator
    default <U extends Comparable<? super U>> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor) {
        
        // 1. 先拿到当前的比较器（this）
        // 2. 再拿到传入的比较器（other）
        // 3. 合并它们
        return (a, b) -> {
            int res = this.compare(a, b);
            if (res != 0) return res; // 如果不相等，直接返回
            return other.compare(a, b); // 如果相等，用下一个规则
        };
    }
}
```

**设计思想**：`Comparator`利用了**装饰者模式**和**函数式编程**。每一个 `thenComparing`都不是修改原对象，而是返回一个包裹了新逻辑的新 `Comparator`。

---

## Q4：为什么 `Callable<V>` 比 `Runnable` 更灵活？

**A：**
- `Runnable`：无返回值（`void run()`），不能抛受检异常
- `Callable<V>`：有返回值（`V call()`），可以抛 `Exception`

在线程池场景中，`Callable` + `Future` 可以获取异步执行结果：
```java
Future<Integer> future = executor.submit(() -> computeResult());
Integer result = future.get();  // 阻塞等待结果
```
