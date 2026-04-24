---
id: qa_124
title: 深拷贝与浅拷贝
tags:
  - 原理型
  - 问答
  - Java/其他核心概念
module: 12_其他核心概念
created: 2026-04-18
---

# 深拷贝与浅拷贝
## Q1：浅拷贝和深拷贝的区别？

**A**：
- **浅拷贝**：创建新对象，基本类型字段复制值，引用类型字段复制引用（共享同一内部对象）
- **深拷贝**：创建新对象，所有字段（包括引用类型）都递归创建新的副本，完全独立
关键区别在于对**引用类型字段**的处理：浅拷贝共享，深拷贝独立。
```java
// 浅拷贝：修改 u2.address 会影响 u1.address
// 深拷贝：修改 u2.address 不影响 u1.address
```

---

## Q2：有哪些实现深拷贝的方式？

**A**：
1. **层层重写 clone()**：每个引用类型都实现 Cloneable 并在 clone 中手动复制
2. **Java 序列化**：通过 ObjectOutputStream → ByteArrayOutputStream → ObjectInputStream 反序列化
3. **JSON 序列化**：Jackson/Gson 的 writeValueAsString → readValue
4. **拷贝构造器/工厂方法**：手动在构造器中创建新对象
实际开发中推荐 JSON 序列化方式，代码简洁且维护成本低。

---

## Q3：浅拷贝会有什么问题？

**A**：修改拷贝对象的引用类型字段会影响原对象（因为共享同一内存地址），可能导致意外的副作用。
```java
User u1 = new User("张三", new Address("杭州"));
User u2 = u1.clone(); // 浅拷贝
u2.getAddress().setCity("北京");
System.out.println(u1.getAddress().getCity()); // "北京"，被意外修改！
```

## Q4：如何正确使用 clone() 方法？

**A**：
1. 类实现 `Cloneable` 标记接口
2. 重写 `clone()` 方法，访问修饰符改为 `public`
3. 在 clone() 中调用 `super.clone()` 并处理异常
```java
public class User implements Cloneable {
    @Override
    public User clone() {
        try {
            return (User) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

---

## Q5：Cloneable 接口有什么设计缺陷？

**A**：
1. **标记接口无方法**：Cloneable 接口本身没有定义任何方法，clone() 定义在 Object 中，违反了面向对象的设计直觉
2. **protected 访问限制**：Object.clone() 是 protected，不能在类外部直接调用
3. **运行时才检查**：如果不实现 Cloneable 就调用 clone()，编译不报错，运行时才抛异常
4. **《Effective Java》建议不使用**：推荐拷贝构造器或拷贝工厂替代

---

## Q6：数组用什么方式拷贝最好？

**A**：
- **`array.clone()`**：最简洁，返回新数组（浅拷贝），推荐用于基本类型数组
- **`Arrays.copyOf()`**：可指定新数组长度
- **`System.arraycopy()`**：性能最高（底层 native 实现），可指定范围
- **`Arrays.copyOfRange()`**：指定范围拷贝
```java
int[] a = {1, 2, 3};
int[] b = a.clone();                    // 推荐
int[] c = Arrays.copyOf(a, a.length);    // 推荐
int[] d = new int[a.length];
System.arraycopy(a, 0, d, 0, a.length);  // 性能最高
```