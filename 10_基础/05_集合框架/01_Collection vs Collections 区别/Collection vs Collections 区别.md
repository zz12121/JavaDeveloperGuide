---
title: Collection vs Collections 区别
tags:
  - Java/集合框架
  - 对比型
module: 05_集合框架
created: 2026-04-18
---

# Collection vs Collections 区别

## 核心对比

| 维度 | Collection | Collections |
|------|-----------|-------------|
| 类型 | **接口**（interface） | **工具类**（class） |
| 所在包 | `java.util` | `java.util` |
| 作用 | 集合框架的**顶层接口**，定义集合的基本操作 | 提供**静态方法**，对集合进行排序、查找、线程安全包装等操作 |
| 能否实例化 | 不能直接 new | 不能 new（私有构造器），通过静态方法调用 |
| 继承关系 | `Iterable` 的子接口 | 继承 `Object` |

## Collection 接口

`Collection` 是所有单列集合的**根接口**：

```
Iterable
  └── Collection
        ├── List    (ArrayList, LinkedList, Vector)
        ├── Set     (HashSet, LinkedHashSet, TreeSet)
        └── Queue   (LinkedList, PriorityQueue, ArrayDeque)
```

核心方法：

```java
boolean add(E e);              // 添加元素
boolean remove(Object o);      // 删除元素
boolean contains(Object o);    // 是否包含
int size();                    // 元素个数
boolean isEmpty();             // 是否为空
Object[] toArray();            // 转数组
Iterator<E> iterator();        // 获取迭代器
```

## Collections 工具类

`Collections` 是操作集合的**工具类**，全部是静态方法：

```java
// 排序
Collections.sort(list);
Collections.sort(list, comparator);
Collections.reverse(list);         // 反转
Collections.shuffle(list);         // 随机打乱

// 查找
Collections.binarySearch(list, key);
Collections.max(list);
Collections.min(list);
Collections.frequency(list, obj);  // 出现次数

// 线程安全包装
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());

// 不可变集合
List<String> immutable = Collections.unmodifiableList(new ArrayList<>());

// 空集合
Collections.emptyList();
Collections.emptyMap();
Collections.emptySet();

// 填充
Collections.fill(list, "default");
Collections.addAll(list, "a", "b", "c");
```

## 注意

- `Map` 不是 `Collection` 的子接口，`Map` 和 `Collection` 是**平级**的
- `Collections` 的 `synchronizedXxx()` 方法只是简单的加 `synchronized` 锁，性能不如并发集合（如 `ConcurrentHashMap`）

## 关联知识点
