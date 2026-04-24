---
title: Object类方法
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

Object 是所有 Java 类的根类，核心方法包括：`getClass()`、`hashCode()`、`equals()`、`clone()`、`toString()`、`finalize()`、`notify()/notifyAll()/wait()`。其中最常被重写的是 equals、hashCode 和 toString。`java.util.Objects` 是 Objects 工具类，提供空值安全的静态方法，JDK 7 引入。

---

## 深度解析

### 1. Object 核心方法一览

| 方法 | 用途 | 是否可重写 |
|------|------|------------|
| `getClass()` | 返回运行时 Class 对象 | 否（final） |
| `hashCode()` | 返回哈希码 | 是 |
| `equals(Object)` | 判断相等 | 是 |
| `clone()` | 克隆对象 | 是（需实现 Cloneable） |
| `toString()` | 返回字符串描述 | 是 |
| `finalize()` | GC 回收前调用（JDK 9+ 已废弃） | 是 |
| `notify()/notifyAll()/wait()` | 线程通信 | 否（final） |

### 2. 常用方法详解

#### getClass()
```java
// 返回运行时类型（非编译时类型）
Animal a = new Dog();
a.getClass() == Dog.class; // true，运行时类型是 Dog
```

#### clone()
```java
// Object.clone() 是 protected，需重写为 public
// 默认是浅拷贝，深拷贝需自行实现
@Override
public Object clone() throws CloneNotSupportedException {
    return super.clone();
}
```

#### toString()
```java
// 默认返回：类名@哈希码的十六进制
// 格式：getClass().getName() + "@" + Integer.toHexString(hashCode())
// 建议重写为有意义的描述
@Override
public String toString() {
    return "User{name='" + name + "', age=" + age + "}";
}
```

#### finalize()
```java
// GC 回收对象前调用，不可靠（不一定执行）
// JDK 9+ 标记为 @Deprecated，不推荐使用
// 推荐 try-with-resources 或 Cleaner 替代
```

### 3. 重写建议

| 方法 | 是否建议重写 | 说明 |
|------|-------------|------|
| `equals()` | ✅ | 当需要内容比较时 |
| `hashCode()` | ✅ | 重写 equals 时必须重写 |
| `toString()` | ✅ | 便于日志和调试 |
| `clone()` | ⚠️ | 谨慎使用，推荐拷贝构造器 |
| `finalize()` | ❌ | 已废弃，不要使用 |

---

### 4. Objects 工具类（JDK 7）

`java.util.Objects` 提供空值安全的静态方法，避免手动判空。

```java
// equals — 空值安全，避免 NPE
Objects.equals(str1, str2);      // 等价于 str1 == null ? str2 == null : str1.equals(str2)

// hashCode — 空值安全
int hash = Objects.hashCode(obj); // 等价于 obj == null ? 0 : obj.hashCode()

// requireNonNull — 参数校验
Objects.requireNonNull(arg, "参数不能为 null");  // 为 null 时抛 NPE 并附带消息

// toString — 空值安全
Objects.toString(obj);           // obj == null 时返回 "null"
Objects.toString(obj, "默认值");  // obj == null 时返回 "默认值"

// compare — 空值安全比较
int result = Objects.compare(a, b, Comparator.naturalOrder());

// isNull / nonNull
Objects.isNull(obj);             // obj == null
Objects.nonNull(obj);            // obj != null
```

> **面试价值**：`Objects.equals()` 是面试常考点，推荐在代码中替代手动 `a != null && a.equals(b)` 的写法。


