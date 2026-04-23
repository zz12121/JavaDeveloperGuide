---
id: card_100
title: 100 String、StringBuilder、StringBuffer 区别
tags:
  - Java/字符串
  - 对比型
module: 09_字符串
created: 2026-04-18
---

# 100 String、StringBuilder、StringBuffer 区别

## 核心结论

`String` 不可变，每次修改创建新对象；`StringBuilder` 可变，非线程安全，性能最好；`StringBuffer` 可变，线程安全（方法加 `synchronized`），性能略低于 `StringBuilder`。单线程拼接字符串用 `StringBuilder`。

## 深度解析

### 三者对比

| 对比项 | String | StringBuilder | StringBuffer |
|--------|--------|---------------|-------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 天然安全 | ❌ 不安全 | ✅ 安全（synchronized） |
| 性能 | 拼接时最差 | 最快 | 略慢于 StringBuilder |
| 使用场景 | 少量字符串操作 | 单线程大量拼接 | 多线程大量拼接 |

### 为什么 String 拼接性能差

```java
// String 拼接：每次 + 都创建新对象
String s = "a" + "b" + "c";  // 编译优化后等价于 "abc"，只创建一个对象
String s2 = "a";
s2 = s2 + "b";  // 运行时拼接：创建 StringBuilder → append → toString → 新 String
```

### StringBuilder 原理

```java
// StringBuilder 内部维护一个可变的 char[] 数组
public final class StringBuilder extends AbstractStringBuilder {
    // 继承 AbstractStringBuilder
}

// AbstractStringBuilder
abstract class AbstractStringBuilder {
    char[] value;       // 可变数组
    int count;          // 当前已用长度

    public AbstractStringBuilder append(String str) {
        ensureCapacityInternal(count + str.length()); // 检查是否需要扩容
        str.getChars(0, str.length(), value, count);  // 复制字符到 value
        count += str.length();
        return this;
    }
}
```

### StringBuffer 的线程安全

```java
// StringBuffer 的方法都加了 synchronized
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}
```

### 性能对比

```java
// String 拼接（慢）
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // 每次循环创建 StringBuilder + String
}

// StringBuilder（快）
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);  // 只在同一个 char[] 上操作
}

// StringBuffer（中等）
StringBuffer sb2 = new StringBuffer();
for (int i = 0; i < 10000; i++) {
    sb2.append(i);  // 有 synchronized 锁开销
}
```

## 关联知识点

