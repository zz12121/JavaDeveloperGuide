---
title: Optional与Stream结合
tags:
  - Java/Stream
  - 场景型
module: 11_Stream API
created: 2026-04-18
---

## 核心结论

`Optional` 是 JDK 8 引入的容器类，用于优雅地处理可能为 null 的值，避免 `NullPointerException`。它与 Stream 配合使用，可以安全地处理空值链和嵌套结构。

---

## 深度解析

### 1. Optional 核心方法

```java
// 创建
Optional.of(value)          // 非 null 值（传入 null 抛 NPE）
Optional.ofNullable(value)  // 允许 null 值
Optional.empty()            // 空实例

// 判断
optional.isPresent()        // 是否有值
optional.isEmpty()          // JDK 11+，是否为空

// 获取
optional.get()              // 有值返回，无值抛 NoSuchElementException（慎用）
optional.orElse(default)    // 有值返回值，无值返回默认值
optional.orElseGet(supplier)// 有值返回值，无值延迟计算默认值
optional.orElseThrow()      // 有值返回值，无值抛异常
optional.or(supplier)       // JDK 9+，无值时切换到另一个 Optional

// 转换
optional.map(fn)            // 值存在时应用函数
optional.flatMap(fn)        // 值存在时应用函数，返回 Optional
optional.filter(predicate)  // 值满足条件时保留

// 消费
optional.ifPresent(consumer)// 有值时执行
optional.ifPresentOrElse(consumer, emptyAction) // JDK 9+
```

### 2. Optional 与 Stream 的协作

#### flatMap 处理嵌套 Optional
```java
// 传统方式：多层 null 检查
String city = null;
if (user != null) {
    Address addr = user.getAddress();
    if (addr != null) {
        city = addr.getCity();
    }
}

// Optional 链式调用
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("未知");
```

#### Stream 中过滤空值
```java
// Optional → Stream（JDK 9+）
optional.stream(); // 将 Optional 转为 0 或 1 个元素的 Stream

// 传统方式：先判断再收集
list.stream()
    .map(Item::getOptionalValue)
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());

// JDK 9+：使用 Optional.stream() + flatMap
list.stream()
    .map(Item::getOptionalValue)
    .flatMap(Optional::stream)
    .collect(Collectors.toList());
```

#### findFirst/findAny 返回 Optional
```java
Optional<User> user = users.stream()
    .filter(u -> u.getName().equals("张三"))
    .findFirst()
    .orElseThrow(() -> new RuntimeException("用户不存在"));
```

### 3. Optional 使用规范

| ✅ 推荐 | ❌ 避免 |
|---------|---------|
| 作为方法返回值 | 作为方法参数 |
| `orElse` / `orElseGet` | 直接调用 `get()` |
| `ifPresent` 消费值 | 用 `isPresent` + `get` 替代 if |
| `map` / `flatMap` 链式调用 | 作为类的字段类型 |
| `Optional.ofNullable` | `Optional.of(null)` |

```java
// ❌ 不推荐：isPresent + get（退化成 null 检查）
if (optional.isPresent()) {
    return optional.get();
}

// ✅ 推荐：链式调用
return optional.orElse(defaultValue);

// ✅ 推荐：ifPresent
optional.ifPresent(System.out::println);
```

---

## 关联知识点

