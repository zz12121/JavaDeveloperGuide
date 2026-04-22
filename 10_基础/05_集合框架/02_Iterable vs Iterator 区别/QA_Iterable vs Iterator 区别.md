---
title: Iterable vs Iterator
tags:
  - Java/集合框架
  - 对比型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Iterable vs Iterator

## Q1：Iterable 和 Iterator 有什么区别？

**A：**
- `Iterable`：表示对象"可遍历"，只有一个方法 `iterator()`，是 `for-each` 语法的前提
- `Iterator`：是实际的迭代器，通过 `hasNext()` 和 `next()` 遍历集合，还支持 `remove()` 删除元素

简单说：`Iterable` 是"声明我能被遍历"，`Iterator` 是"实际执行遍历"。

---

## Q2：for-each 循环的底层原理是什么？

**A：**
`for-each` 是语法糖，编译后转换为调用 `iterator()`：

```java
for (String s : list) { ... }

// 编译后等价于
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    ...
}
```

前提条件：目标对象必须实现 `Iterable` 接口。

---

## Q3：遍历时删除元素，为什么用 Iterator.remove() 而不是 list.remove()？

**A：**
- `list.remove()` 会修改集合的 `modCount`，但 `Iterator` 检测到 `modCount` 变化会抛出 `ConcurrentModificationException`（fail-fast）
- `Iterator.remove()` 在删除后会同步更新 `expectedModCount`，不会触发异常

---

## Q4：Iterator 有哪些特点？

**A：**
1. **单向遍历**：只能从前往后，不能回退
2. **fail-fast**：多数集合的 Iterator 在遍历时检测到结构性修改会抛 CME
3. **安全删除**：`remove()` 是遍历中删除元素的唯一安全方式
