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

### JDK 9+ Compact Strings（LATIN1 vs UTF16 编码）

JVM 从 JDK 9 开始引入 **Compact Strings** 优化：若字符串只包含 Latin-1 字符（一个字符只需 1 字节），使用 `byte[]` + `coder=LATIN1`；包含其他 Unicode 字符时使用 `byte[]` + `coder=UTF16`。

**为什么从 char[] 改为 byte[]？**

| 维度 | JDK 8（char[]） | JDK 9+（byte[]） |
|------|:---:|:---:|
| Latin-1 字符 | 2 字节/字符（char） | 1 字节/字符（byte） |
| 中文字符 | 2 字节/字符 | 2 字节/字符（UTF16，需要2个byte） |
| 普通英文文本内存 | 100% | 约 50% |
| 源码复杂度 | 低 | 高（需要 coder 判断） |

**coder 的作用**：

```java
// coder = 0（LATIN1）：每个 byte 直接存储一个字符
// coder = 1（UTF16）：每两个 byte 存储一个字符（Java char）
// coder = 2（JDK 15+，UTF-8）：紧凑的字节存储

public final class String {
    private final byte[] value;
    private final byte coder;  // COMPACT_STRINGS 开关，JVM 启动参数可关闭

    // 根据 coder 选择解码方式
    char charAt(int index) {
        if (coder == LATIN1) {
            return (char)(value[index] & 0xff);  // 单字节 Latin-1
        }
        return StringUTF16.getChar(value, index);  // 双字节 UTF16
    }
}
```

**与不可变性的关系**：`final byte[] value` + `final byte coder` 保证了编码和内容的双重不可变。即使 JVM 内部会在 Latin-1 和 UTF16 之间做视图切换（如 `getBytes()`），String 对象本身永远不变。

**JVM 参数**：
```bash
# 禁用 Compact Strings（使用旧的 char[] 实现）
-XX:-CompactStrings
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
