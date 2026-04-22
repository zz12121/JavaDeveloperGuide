---
title: fail-fast 与 fail-safe 机制
tags:
  - Java/集合框架
  - 原理型
module: 05_集合框架
created: 2026-04-18
---

# fail-fast 与 fail-safe 机制

## 核心结论

**fail-fast**（快速失败）和 **fail-safe**（安全失败）是集合框架中两种不同的并发修改处理策略。
fail-fast 在检测到并发修改时**立即抛出 `ConcurrentModificationException`**；fail-safe 则在遍历**副本**上操作，不会抛出异常，但不保证看到最新修改。

---

## 深度解析

### 1. fail-fast 机制

**原理**：集合内部维护一个 `modCount`（修改计数器），Iterator 在创建时记录 `expectedModCount`。每次迭代前检查两者是否一致，不一致则抛出 `ConcurrentModificationException`。

```java
// ArrayList.iterator() 源码关键逻辑
private class Itr implements Iterator<E> {
    int expectedModCount = modCount;  // 创建时快照

    public E next() {
        checkForComodification();     // 每次调用检查
        // ...
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

**触发场景**：
- 单线程：遍历时调用集合自身的 `add()`/`remove()`（非 Iterator 的方法）
- 多线程：一个线程遍历，另一个线程修改集合

### 2. fail-safe 机制

**原理**：不直接操作原集合，而是先**拷贝一份**（或使用线程安全的数据结构），在副本上遍历。修改原集合不影响遍历。

**实现方式**：
- `java.util.concurrent` 包下的集合（如 `ConcurrentHashMap`、`CopyOnWriteArrayList`）
- 使用 `Collections.synchronizedList()` 包装

```java
// CopyOnWriteArrayList：写时复制
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");

// 遍历时可以安全修改
for (String s : list) {
    list.add("C");  // ✅ 不抛异常，但遍历中看不到新元素
}
```

### 3. 对比

| 对比项 | fail-fast | fail-safe |
|--------|-----------|-----------|
| 异常 | 抛出 `ConcurrentModificationException` | 不抛异常 |
| 遍历对象 | 原集合 | 集合的副本 |
| 内存开销 | 低 | 高（需要拷贝） |
| 数据一致性 | 遍历时能看到实时修改 | 遍历的是快照，看不到最新修改 |
| 代表集合 | `ArrayList`、`HashMap`、`HashSet` | `ConcurrentHashMap`、`CopyOnWriteArrayList` |
| 适用场景 | 单线程或外层加同步 | 多线程并发读写 |

### 4. 正确的删除方式

```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

// ❌ 错误：遍历时用集合的 remove → ConcurrentModificationException
for (String s : list) {
    if (s.equals("B")) list.remove(s);
}

// ✅ 方式一：使用 Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove();
}

// ✅ 方式二：使用 removeIf（JDK 8）
list.removeIf(s -> s.equals("B"));

// ✅ 方式三：使用 Stream 过滤
list = list.stream().filter(s -> !s.equals("B")).collect(Collectors.toList());
```

---

## 代码示例

```java
// fail-fast 演示
public class FailFastDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("A");
        list.add("B");
        list.add("C");

        for (String s : list) {
            System.out.println(s);
            if (s.equals("A")) {
                list.add("D");  // ⚠️ 抛出 ConcurrentModificationException
            }
        }
    }
}
```

---

## 关联知识点
