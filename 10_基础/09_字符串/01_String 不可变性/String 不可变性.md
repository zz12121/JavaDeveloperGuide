---
title: String 不可变性
tags:
  - Java/字符串
  - 原理型
module: 09_字符串
created: 2026-04-18
---

# String 不可变性

## 核心结论

String 被设计为**不可变（Immutable）**：一旦创建，其内容不能被修改。底层通过 `final char[] value`（JDK 8）或 `final byte[] value`（JDK 9+）存储数据，`final` 修饰保证引用不可变，私有化且无 setter 方法保证值不可变。

## 深度解析

### String 不可变的实现原理

```java
// JDK 8
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char[] value;  // final 修饰
    private final int hash;      // 缓存哈希值

    // 构造器复制数组，不直接引用外部数组
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
}
```

```java
// JDK 9+（Compact Strings 优化）
public final class String {
    private final byte[] value;   // final 修饰，byte 数组节省空间
    private final byte coder;     // LATIN1=0 / UTF16=1
}
```

不可变的四层保障：
1. **`final` 修饰类**：String 类不能被继承，防止子类破坏不可变性
2. **`final` 修饰 value 数组**：value 引用不可指向其他数组
3. **`private` 修饰 value**：外部无法直接访问 value 数组
4. **没有 setter 方法**：没有任何方法能修改 value 中的内容

### 为什么设计为不可变

| 原因 | 说明 |
|------|------|
| **字符串常量池** | 不可变对象可以安全共享，多个引用指向同一个 String 对象 |
| **线程安全** | 不可变对象天然线程安全，无需同步 |
| **安全** | HashMap 的 key、网络连接参数、文件路径等不会意外被修改 |
| **hashCode 缓存** | String 的 hashCode 只计算一次并缓存，HashMap 查找高效 |
| **防止反射修改** | 虽然可以通过反射修改 value 数组内容，但这是不推荐的行为 |

### 不可变带来的影响

```java
// "修改" String 实际上是创建了新对象
String s1 = "hello";
String s2 = s1.concat(" world"); // s1 不变，s2 是新对象
System.out.println(s1); // hello（未改变）
System.out.println(s2); // hello world（新对象）

// 频繁拼接会产生大量临时对象
String s = "";
for (int i = 0; i < 1000; i++) {
    s += i;  // 每次循环创建一个新 String 对象
}
```

## 关联知识点
