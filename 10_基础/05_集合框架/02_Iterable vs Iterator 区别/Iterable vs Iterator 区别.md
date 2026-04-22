---
title: Iterable vs Iterator 区别
tags:
  - Java/集合框架
  - 对比型
module: 05_集合框架
created: 2026-04-18
---

# Iterable vs Iterator 区别

## 核心对比

| 维度 | Iterable | Iterator |
|------|---------|---------|
| 类型 | **接口** | **接口** |
| 作用 | 表示"**可迭代的**"，是 `for-each` 的前提 | 表示"**迭代器**"，提供遍历集合的能力 |
| 方法数量 | 只有 1 个核心方法 | 4 个核心方法 |
| 调用关系 | 被集合实现，返回 Iterator | 由集合创建，实际执行遍历 |
| 能否删除元素 | 不能 | 可以（通过 `remove()`） |

## Iterable 接口

```java
public interface Iterable<T> {
    Iterator<T> iterator();  // 唯一的抽象方法
}
```

- 任何实现了 `Iterable` 的类都可以使用 **`for-each`** 语法
- `Collection` 继承了 `Iterable`，所以所有集合类都支持 `for-each`

```java
// for-each 语法糖，编译后实际调用 iterator()
for (String s : list) {
    System.out.println(s);
}

// 等价于
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```

JDK 8 后 `Iterable` 新增了默认方法：

```java
default void forEach(Consumer<? super T> action);  // 遍历
default Spliterator<T> spliterator();               // 可分割迭代器
```

## Iterator 接口

```java
public interface Iterator<E> {
    boolean hasNext();  // 是否还有下一个元素
    E next();           // 返回下一个元素
    default void remove();          // 删除当前元素（可选操作）
    default void forEachRemaining(Consumer<? super E> action);  // JDK 8
}
```

### Iterator 的特点

1. **单向遍历**：只能从头到尾，不能回退
2. **安全删除**：`remove()` 是唯一安全的在遍历时删除元素的方式

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("target")) {
        it.remove();  // 安全删除当前元素
    }
}
```

### 为什么 for 循环中不能直接 remove

```java
// 错误：会抛出 ConcurrentModificationException
for (String s : list) {
    if (s.equals("target")) {
        list.remove(s);  // 修改了结构，但 Iterator 不知道
    }
}

// 正确：使用 Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("target")) {
        it.remove();
    }
}
```

## 关系图

```
Iterable（可迭代的）
  └── Collection
        └── ArrayList  ── iterator() ──→  Iterator（迭代器）
                                             ├── hasNext()
                                             ├── next()
                                             └── remove()
```

## 关联知识点
