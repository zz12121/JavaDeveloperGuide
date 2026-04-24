---
title: 深拷贝与浅拷贝
tags:
  - 原理型
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

## 核心结论

- **浅拷贝**：创建新对象，但引用类型字段仍指向原对象的引用（共享内部对象）
- **深拷贝**：创建新对象，所有引用类型字段也递归创建新副本（完全独立）

---

## 深度解析

### 1. 浅拷贝

```
原对象                    浅拷贝对象
┌──────────┐            ┌──────────┐
│ name: "A"│            │ name: "A"│  ← String 不可变，无影响
│ age: 20  │            │ age: 20  │  ← 基本类型，独立
│ addr: ───┼──→ [内存地址] ←──┼── addr:  │  ← 引用类型，共享同一对象！
└──────────┘            └──────────┘
```

```java
class User implements Cloneable {
    String name;
    int age;
    Address address; // 引用类型

    @Override
    public User clone() {
        return (User) super.clone(); // 浅拷贝
    }
}

User u1 = new User("张三", 18, new Address("杭州"));
User u2 = u1.clone();
u2.address.setCity("北京"); // u1.address.city 也变成了"北京"！
```

### 2. 深拷贝

```
原对象                    深拷贝对象
┌──────────┐            ┌──────────┐
│ name: "A"│            │ name: "A"│
│ age: 20  │            │ age: 20  │
│ addr: ───┼──→ [Address1]  │ addr: ───┼──→ [Address2]  ← 独立的副本
└──────────┘            └──────────┘
```

### 3. 深拷贝实现方式

#### 方式一：重写 clone（层层重写）
```java
class Address implements Cloneable {
    String city;
    @Override
    public Address clone() { return (Address) super.clone(); }
}

class User implements Cloneable {
    Address address;
    @Override
    public User clone() {
        User copy = (User) super.clone();
        copy.address = address.clone(); // 深拷贝引用字段
        return copy;
    }
}
```

#### 方式二：序列化（推荐）
```java
// 利用 ObjectOutputStream/ObjectInputStream 实现深拷贝
@SuppressWarnings("unchecked")
public static <T extends Serializable> T deepCopy(T obj) {
    try {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        new ObjectOutputStream(bos).writeObject(obj);
        return (T) new ObjectInputStream(
            new ByteArrayInputStream(bos.toByteArray())).readObject();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

#### 方式三：JSON 序列化
```java
// 利用 Jackson/Gson
String json = objectMapper.writeValueAsString(original);
MyClass copy = objectMapper.readValue(json, MyClass.class);
```

### 4. 对比

| 方式 | 优点 | 缺点 |
|------|------|------|
| 浅拷贝 clone | 简单 | 引用字段共享 |
| 层层重写 clone | 不需要额外依赖 | 维护麻烦，链式结构复杂 |
| Java 序列化 | 通用，无需手动处理 | 性能较差，要求实现 Serializable |
| JSON 序列化 | 灵活，可跨语言 | 需要引入第三方库 |

---

# clone方法与Cloneable接口
## 核心结论

`clone()` 是 Object 的 protected 方法，默认执行浅拷贝。要使用 clone，类必须实现 `Cloneable` 接口（标记接口），否则会抛出 `CloneNotSupportedException`。

---

## 深度解析

### 1. 使用步骤

```java
// 1. 实现 Cloneable 标记接口
// 2. 重写 clone() 方法，改为 public
// 3. 调用 super.clone()
public class User implements Cloneable {
    private String name;
    private int age;

    @Override
    public User clone() {
        try {
            return (User) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // 不会发生，因为已实现 Cloneable
        }
    }
}
```

### 2. Cloneable 的设计问题

|问题|说明|
|---|---|
|接口中没有 clone() 方法|标记接口，本身无方法声明|
|Object.clone() 是 protected|不能在外部类直接调用 obj.clone()|
|返回 Object 类型|需要强制类型转换|
|依赖运行时检查|不实现接口只重写方法，运行时才抛异常|

### 3. 数组的拷贝

```java
// clone() 是数组的浅拷贝最佳方式
int[] original = {1, 2, 3};
int[] copy = original.clone(); // 新数组，元素值相同
copy[0] = 99;
System.out.println(original[0]); // 1，不受影响（基本类型）

// 引用类型数组仍然是浅拷贝
User[] users = {new User("张三"), new User("李四")};
User[] copy = users.clone();
copy[0].setName("王五"); // users[0].name 也变成 "王五"
```

### 4. 更好的替代方案

```java
// ✅ 推荐：拷贝构造器
public User(User other) {
    this.name = other.name;
    this.age = other.age;
}

// ✅ 推荐：静态工厂
public static User copyOf(User other) {
    return new User(other.name, other.age);
}
```

## 关联知识点