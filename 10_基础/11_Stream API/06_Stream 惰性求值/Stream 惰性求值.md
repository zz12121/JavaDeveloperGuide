---
title: Stream惰性求值
tags:
  - Java/Stream
  - 原理型
module: 11_Stream API
created: 2026-04-18
---

## 核心结论

Stream 的中间操作是**惰性求值**（Lazy Evaluation）的——它们不会立即执行，而是记录操作轨迹。只有当终端操作被调用时，整个流水线才会被**一次性执行**，且数据是**逐元素流过**整个流水线，而非先完成一个操作再进行下一个。

---

## 深度解析

### 1. 什么是惰性求值

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);

// 没有终端操作，filter 和 map 都不会执行
nums.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n > 2;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n * 2;
    });
// 输出：（无任何输出）
```

```java
// 加上终端操作后，整个流水线才执行
nums.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n > 2;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n * 2;
    })
    .count();
// 输出：
// filter: 1
// filter: 2
// filter: 3
// map: 3
// filter: 4
// map: 4
// filter: 5
// map: 5
```

### 2. 执行机制：逐元素流过

**不是**先过滤所有元素，再映射所有元素。

**而是**每个元素依次通过 filter → map → 终端操作：

```
元素1 → filter(不通过) → 跳过
元素2 → filter(不通过) → 跳过
元素3 → filter(通过) → map → 终端操作收集
元素4 → filter(通过) → map → 终端操作收集
元素5 → filter(通过) → map → 终端操作收集
```

这种机制带来的好处：
- **不需要中间集合**：不在内存中存储中间结果
- **短路优化**：`findFirst` 等短路终端操作可以在找到结果后立即停止

### 3. 短路操作与惰性求值的配合

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);

// findFirst 是短路操作，找到第一个满足条件的就停
Optional<Integer> result = nums.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n > 2;
    })
    .findFirst();
// 输出：
// filter: 1
// filter: 2
// filter: 3
// 注意：4 和 5 不会被处理！
```

### 4. 惰性求值 vs 及早求值

| 维度 | Stream（惰性求值） | 传统 for 循环（及早求值） |
|------|---------------------|---------------------------|
| 执行时机 | 终端操作触发 | 立即执行 |
| 中间结果 | 不存储，逐元素流过 | 可能需要中间集合 |
| 短路优化 | 自动支持 | 需手动 break |
| 代码可读性 | 声明式，描述"做什么" | 命令式，描述"怎么做" |
| 调试难度 | 较高（peek 辅助） | 较低 |

### 5. 实际影响

```java
// 惰性求值可以避免不必要的计算
// 即使 filter 前面有昂贵的 map 操作，不满足条件的元素不会执行到 map

List<String> longList = ...; // 100万个元素

// map 只会在 filter 通过的元素上执行
longList.stream()
    .filter(s -> s.length() > 10)  // 先过滤掉大部分
    .map(s -> expensiveTransform(s))  // 只对少量元素执行昂贵操作
    .findFirst();                     // 找到第一个就停
```

---

## 关联知识点
