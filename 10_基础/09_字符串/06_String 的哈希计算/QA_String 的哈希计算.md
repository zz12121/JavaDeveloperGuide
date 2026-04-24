---
title: String 的哈希计算
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String 的哈希计算

## Q：String 的 hashCode() 是怎么计算的？

**A：** 使用乘法哈希算法：`h = 31 * h + char[i]`，展开后为 `s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`。
源码关键点：
1. **延迟计算**：首次调用 `hashCode()` 时才遍历字符数组计算
2. **缓存**：计算结果存入 `hash` 字段，后续调用直接返回缓存值
3. String 不可变保证了 `hash` 缓存始终有效
```java
String s = "hello";
s.hashCode();  // 首次：遍历计算并缓存
s.hashCode();  // 后续：直接返回缓存，O(1)
```

## Q：为什么 String 的 hashCode 乘数选择 31？

**A：**
1. **31 是质数**：减少哈希碰撞
2. **31 是奇数**：避免乘法信息丢失
3. **31 = 2^5 - 1**：`31 * h` 可以优化为位运算 `(h << 5) - h`，JVM 对此有专门优化
4. **经验验证**：经过大量实践验证效果良好

## Q：String 的 hashCode 会不会碰撞？

**A：** 会的。不同字符串可能有相同的 hashCode：
```java
"Aa".hashCode();  // 2112
"BB".hashCode();  // 2112
```
但 `equals()` 不同，HashMap 中通过 `hashCode()` 定位桶，再用 `equals()` 精确匹配，所以不会冲突。
