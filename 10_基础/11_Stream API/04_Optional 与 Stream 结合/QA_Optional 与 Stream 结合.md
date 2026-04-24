---
title: Optional与Stream结合
tags:
  - Java/Stream
  - 场景型
  - 问答
module: 11_Stream API
created: 2026-04-18
---

# Optional与Stream结合
## Q1：Optional 的主要用途是什么？

**A**：Optional 是一个**容器对象**，用于表示一个值可能存在也可能不存在。核心目的是：
1. **替代 null 检查**，避免 NPE
2. **明确表达语义**，方法返回 Optional 比返回 null 更清晰地表达了"可能没有值"的意图
3. **强制调用方处理空值情况**，编译器层面提醒开发者
```java
// 传统：调用方容易忘记判空
public User findUser(String name) { return null; }

// 推荐：Optional 明确告知可能为空
public Optional<User> findUser(String name) { return Optional.empty(); }
```

---

## Q2：orElse 和 orElseGet 的区别？

**A**：
- **`orElse(value)`**：无论 Optional 是否有值，默认值都会被**立即计算**
- **`orElseGet(Supplier)`**：仅在 Optional 为空时才**延迟计算**默认值
```java
// orElse：即使有值，logError() 也会执行（浪费资源）
optional.orElse(logError());

// orElseGet：有值时 logError() 不会执行
optional.orElseGet(() -> logError());
```
> **性能陷阱**：当默认值的构造成本较高（如数据库查询、复杂计算）时，必须使用 `orElseGet`。

---

## Q3：Optional 不应该在哪些场景使用？

**A**：
1. **不要用作方法参数** — 增加调用复杂度，应改用方法重载
2. **不要用作类的字段** — Optional 未实现 Serializable，且占更多内存
3. **不要用 isPresent + get 替代 if-null** — 这违背了 Optional 的设计初衷
4. **不要用 Optional.of(null)** — 会直接抛 NPE，应使用 `ofNullable`
```java
// ❌ 类字段
private Optional<String> name; // 不要这样做

// ✅ 类字段用 null 或空字符串
private String name;

// ❌ 方法参数
public void process(Optional<String> input) { }

// ✅ 方法重载
public void process(String input) { }
public void process() { }
```

---

## Q4：如何用 Optional 简化多层嵌套判空？

**A**：使用 `map` + `orElse` 链式调用，替代传统的逐层 null 检查：
```java
// 传统方式（3层判断）
String city = null;
if (order != null) {
    User user = order.getUser();
    if (user != null) {
        Address addr = user.getAddress();
        if (addr != null) {
            city = addr.getCity();
        }
    }
}

// Optional 链式（一行搞定）
String city = Optional.ofNullable(order)
    .map(Order::getUser)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("未知城市");
```
