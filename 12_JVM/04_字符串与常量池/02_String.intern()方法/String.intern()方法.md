
# String.intern() 方法

## 先说结论

`String.intern()` 将字符串放入字符串常量池，返回常量池中的引用。JDK6 常量池在永久代，JDK7+ 在堆中，行为略有不同。

## 深度解析

### intern() 方法原理

```java
public native String intern();

// 内部逻辑：
// 1. 如果常量池已有该字符串，返回常量池引用
// 2. 如果没有，将字符串加入常量池，返回常量池引用
```

### JDK6 vs JDK7+ 行为差异

| JDK 版本 | 常量池位置 | intern() 行为 |
|----------|------------|---------------|
| **JDK6** | 永久代 | 字符串复制到永久代，可能 OOM |
| **JDK7+** | 堆 | 字符串引用复制到常量池，更安全 |

### 经典面试题

```java
// JDK7+ 结果
String s1 = new String("a") + new String("b");
s1.intern();
String s2 = "ab";
System.out.println(s1 == s2); // JDK7+: true, JDK6: false
```

**解释**：JDK7+ intern() 存的是引用，s1 实际就是 "ab"，所以 s1 和 s2 指向同一对象。

## 易错点/踩坑

- ❌ JDK6 用 intern() 可能导致永久代 OOM
- ❌ JDK6 和 JDK7+ 的 intern() 行为有重大区别
- ❌ new String("a") 会创建 2 个对象（字符串常量池 + 堆）

## 代码示例

```java
// 示例1
String s1 = "hello";
String s2 = new String("hello");
System.out.println(s1 == s2); // false

// 示例2
String s3 = s2.intern(); // 返回常量池引用
System.out.println(s1 == s3); // true

// 示例3：字符串拼接优化
String a = "a" + "b"; // 编译期优化为 "ab"
String b = "ab";
System.out.println(a == b); // true
```

## 图解/流程

```
┌─────────────────────────────────────────┐
│         String.intern() 流程            │
├─────────────────────────────────────────┤
│  调用 intern()                          │
│      ↓                                  │
│  检查字符串常量池                        │
│      ↓                                  │
│  ┌─────────────────┐                   │
│  │ 已存在？        │ ─── 是 ──→ 返回常量池引用│
│  └────────┬────────┘                   │
│           │ 否                         │
│           ↓                            │
│  将字符串加入常量池                     │
│      ↓                                  │
│  返回常量池引用                         │
└─────────────────────────────────────────┘
```

## 关联知识点