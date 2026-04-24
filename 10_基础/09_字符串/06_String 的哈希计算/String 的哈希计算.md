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

# hashCode与equals关系
## 核心结论

**hashCode 和 equals 必须同时重写**：如果两个对象 equals 相等，它们的 hashCode 必须也相等。反过来，hashCode 相等的两个对象，equals 不一定相等（哈希冲突）。

---

## 深度解析

### 1. 约定规则

|规则|说明|
|---|---|
|一致性|同一对象多次调用 hashCode 返回相同值|
|equals 为真 → hashCode 相同|两个 equals 的对象必须有相同 hashCode|
|hashCode 相同 → equals 不一定为真|不同对象可能哈希冲突|
|equals 为假 → hashCode 可以相同也可以不同|但不同 hashCode 能提高哈希表性能|

### 2. 为什么重写 equals 必须重写 hashCode

HashMap 的存取流程依赖 hashCode 先定位桶，再用 equals 精确匹配：
```java
// 如果只重写 equals 不重写 hashCode：
User u1 = new User("张三", 18);
User u2 = new User("张三", 18);

u1.equals(u2); // true（内容相同）

Map<User, String> map = new HashMap<>();
map.put(u1, "value");
map.get(u2);    // null！因为 u2 的 hashCode 与 u1 不同，定位到不同桶
```

### 3. 重写规范

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    User user = (User) o;
    return age == user.age && Objects.equals(name, user.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

### 4. 常见问题

#### 重写 equals 但不重写 hashCode → HashMap 等集合出 bug
#### hashCode 依赖可变字段 → 放入 HashMap 后修改字段导致找不到

```java
// 实体类放入 Map 后不要修改参与 hashCode 计算的字段
Set<User> set = new HashSet<>();
User u = new User("张三", 18);
set.add(u);
u.setAge(20);          // 修改后 hashCode 变了
set.contains(u);       // false！
```

### 5. 核心记忆口诀

> **equals 相等 → hashCode 必须相等**（强制）  
> **hashCode 相等 → equals 不一定相等**（允许哈希冲突）  
> **equals 不相等 → hashCode 最好不相等**（性能优化）


## 关联知识点
