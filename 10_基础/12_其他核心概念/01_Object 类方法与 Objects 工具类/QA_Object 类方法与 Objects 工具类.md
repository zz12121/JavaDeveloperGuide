---
title: Object类方法
tags:
  - 原理型
  - 问答
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# Object类方法
## Q1：Object 类有哪些方法？

**A**：Object 类是所有 Java 类的根类，核心方法包括：
- `getClass()` — 获取运行时 Class 对象（final，不可重写）
- `hashCode()` — 返回哈希码
- `equals(Object)` — 判断是否相等
- `clone()` — 克隆对象（protected，需实现 Cloneable）
- `toString()` — 返回字符串表示
- `finalize()` — GC 回收前回调（已废弃）
- `notify()/notifyAll()/wait()` — 线程通信（final）

---

## Q2：为什么 clone() 方法不推荐使用？

**A**：
1. **默认浅拷贝**：`Object.clone()` 只做浅拷贝，引用类型字段仍共享同一对象
2. **使用复杂**：需实现 Cloneable 接口、重写 clone() 为 public、处理 CloneNotSupportedException
3. **未调用构造器**：clone 不经过构造方法，可能违反不变性约束
4. **可替代方案更好**：拷贝构造器、序列化/反序列化、BeanUtils.copyProperties
```java
// 推荐替代：拷贝构造器
public User(User other) {
    this.name = other.name;
    this.age = other.age;
}

// 或静态工厂
public static User copyOf(User other) {
    return new User(other.name, other.age);
}
```
