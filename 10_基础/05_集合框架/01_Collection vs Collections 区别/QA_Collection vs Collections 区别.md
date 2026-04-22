---
title: Collection vs Collections
tags:
  - Java/集合框架
  - 对比型
  - 问答
module: 05_集合框架
created: 2026-04-18
---

# Collection vs Collections

## Q1：Collection 和 Collections 有什么区别？

**A：**
- `Collection` 是集合框架的**顶层接口**，`List`、`Set`、`Queue` 都继承自它
- `Collections` 是一个**工具类**，提供各种静态方法来操作集合（排序、查找、反转、线程安全包装等）

```java
Collection<String> c = new ArrayList<>();  // Collection 是接口
Collections.sort(list);                     // Collections 是工具类
```

---

## Q2：Collections 常用方法有哪些？

**A：**

| 方法分类 | 方法 | 说明 |
|---------|------|------|
| 排序 | `sort(list)` / `sort(list, comp)` | 自然序 / 自定义序 |
| 查找 | `binarySearch(list, key)` | 二分查找（必须先排序） |
| 反转 | `reverse(list)` | 反转列表 |
| 打乱 | `shuffle(list)` | 随机打乱 |
| 不可变 | `unmodifiableList(list)` | 返回只读视图 |
| 线程安全 | `synchronizedList(list)` | 返回线程安全包装 |

---

## Q3：Map 是 Collection 的子接口吗？

**A：**
不是。`Map` 和 `Collection` 是平级的，`Map` 没有继承 `Collection`。虽然可以通过 `map.entrySet()` 获取 `Collection` 视图，但 `Map` 本身不在 `Collection` 体系内。
