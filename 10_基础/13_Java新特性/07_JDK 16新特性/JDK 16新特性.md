---
title: JDK 16新特性
tags:
  - Java/新特性
  - 原理型
module: 13_Java新特性
created: 2026-04-18
---

# Records 记录类

## 核心结论

Records（记录类）是 JDK 16 正式引入的特性（JDK 14 预览），用于快速定义**不可变数据载体类**。编译器自动生成构造方法、访问器方法、`equals()`、`hashCode()`、`toString()`。Records 是 `final` 的，不能被继承，适合替代 Lombok 的 `@Data`（只读场景）。

---

## 深度解析

### 1. 基本定义

```java
// 传统 POJO（约 50+ 行）
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { return Objects.hash(x, y); }
    @Override public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}

// Record（1 行）
public record Point(int x, int y) {}
```

### 2. 编译器自动生成的内容

| 自动生成 | 说明 |
|----------|------|
| `private final` 字段 | 每个组件对应一个 |
| 规范构造方法 | `public Point(int x, int y)` |
| 访问器方法 | `public int x()`、`public int y()`（**不是 getX()**） |
| `equals()` | 基于所有组件值判断相等 |
| `hashCode()` | 基于所有组件值计算哈希 |
| `toString()` | 返回 `Point[x=1, y=2]` 格式 |

### 3. 自定义构造方法

```java
public record Range(int min, int max) {
    // 紧凑构造方法：验证逻辑，不重复参数列表
    public Range {
        if (min > max) throw new IllegalArgumentException("min > max");
    }

    // 自定义工厂方法
    public static Range of(int max) {
        return new Range(0, max);
    }
}
```

### 4. 自定义访问器方法

```java
public record User(String name, String email) {
    // 覆盖自动生成的访问器方法
    @Override
    public String email() {
        return email.toLowerCase();  // 返回处理后的值
    }
}
```

### 5. Record 的限制

| 限制 | 说明 |
|------|------|
| 不可继承 | Record 隐式 `final`，不能 `extends` 其他类 |
| 可以实现接口 | `record Point(int x, int y) implements Serializable {}` |
| 字段不可变 | 组件字段是 `private final`，只能通过构造方法设置 |
| 不能有实例字段 | 只有组件字段，不能额外声明 `private String extra;` |
| 不能变长参数 | Record 组件不能是 varargs |

### 6. 与 Lombok @Data 对比

| 对比项 | Record | Lombok @Data |
|--------|--------|-------------|
| 不可变性 | ✅ 天然不可变 | ❌ 可变（有 setter） |
| JDK 依赖 | JDK 16+ | 任何 JDK + Lombok 依赖 |
| equals/hashCode | ✅ 标准实现 | ✅ 标准实现 |
| 可读性 | ✅ 语法原生 | ⚠️ 需要注解处理器 |
| 继承 | 不能被继承 | 可以被继承 |

### 7. 实际应用场景

```java
// DTO 传输对象
public record UserDTO(String name, String email, int age) {}

// 方法多返回值
public record Pair<K, V>(K key, V value) {}

// 配置项
public record DatabaseConfig(String url, String username, int poolSize) {}

// Map Entry 替代
public record Entry<K, V>(K key, V value) implements Map.Entry<K, V> {
    @Override public K getKey() { return key; }
    @Override public V getValue() { return value; }
    @Override public V setValue(V value) { throw new UnsupportedOperationException(); }
}
```

---

## 关联知识点

