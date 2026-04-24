---
title: String 的哈希计算
tags:
  - Java/字符串
  - 原理型
module: 09_字符串
created: 2026-04-18
---

# String 的哈希计算

## 核心结论

String 的 `hashCode()` 使用**延迟计算 + 缓存**策略：首次调用时计算哈希值并缓存到 `hash` 字段，后续直接返回缓存值。算法采用乘法哈希：`s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`。

## 深度解析

### hashCode 源码

```java
public class String {
    private final char[] value;  // JDK 8
    private int hash;            // 默认 0，缓存哈希值

    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;
            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
}
```

### 为什么选择 31 作为乘数

| 原因 | 说明 |
|------|------|
| **奇数** | 奇数乘法不会丢失信息，偶数可能导致哈希冲突增加 |
| **质数** | 减少哈希冲突 |
| **31 = 2^5 - 1** | 优化为位移运算：`31 * h = (h << 5) - h`，性能好 |
| **经验验证** | Java 标准库广泛使用 31，经过大量实践验证效果良好 |

### 缓存策略的意义

```java
String s = "hello";
int h1 = s.hashCode();  // 首次调用，遍历计算
int h2 = s.hashCode();  // 直接返回缓存的 hash 字段，O(1)
```

- String 的不可变性保证了 `hash` 缓存始终有效
- HashMap 中 String 作为 key 时，查找效率高（hashCode 只算一次）

### 空字符串的哈希值

```java
"".hashCode();  // 返回 0
```

### 哈希碰撞示例

```java
// 两个不同的字符串可能有相同的 hashCode
"Aa".hashCode();  // 2112
"BB".hashCode();  // 2112
// 但 equals() 不同，HashMap 中仍能正确区分
```

## 关联知识点
