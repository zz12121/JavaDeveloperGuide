---
title: 泛型与反射结合
tags:
  - Java/泛型
  - 原理型
  - 问答
module: 04_泛型
created: 2026-04-18
---

# 泛型与反射结合

## Q1：泛型擦除后，能通过反射获取到泛型类型信息吗？

**A：**
**可以部分获取**。虽然运行时实例上的类型参数被擦除，但**声明层面的泛型类型信息**（类定义、方法签名、字段声明上的泛型）会保存在字节码的 `Signature` 属性中，可通过反射读取：

```java
// 可以获取：字段/方法上声明的泛型类型
Field f = Container.class.getDeclaredField("items");  // List<String>
Type t = f.getGenericType();  // 获得 List<String>

// 不可以获取：运行时实例的类型参数
List<String> list = new ArrayList<>();
// list 本身无法告诉你它是 List<String>
```

---

## Q2：Jackson 是如何反序列化泛型类型的（如 `List<User>`）？

**A：**
通过**超类类型令牌（Super Type Token）**：

```java
// 创建 TypeReference 的匿名子类
// 泛型参数 List<User> 会被记录到匿名类的 Signature 字节码中
new TypeReference<List<User>>() {}

// Jackson 内部读取
ParameterizedType type = (ParameterizedType) 
    typeReference.getClass().getGenericSuperclass();
// 获得实际类型参数 List<User>，然后进行精确反序列化
```

---

## Q3：Spring MVC 是如何知道方法返回的 `List<User>` 中泛型是 `User` 的？

**A：**
Spring 通过反射读取方法的**泛型返回类型**：

```java
Method method = controller.getClass().getMethod("findAll");
Type returnType = method.getGenericReturnType();  // List<User>

if (returnType instanceof ParameterizedType) {
    Type[] args = ((ParameterizedType) returnType).getActualTypeArguments();
    // args[0] = User.class → 用于 JSON 序列化
}
```

这也是为什么 Spring 的 `ResponseBody` 能正确序列化泛型类型。

---

## Q4：如何在抽象基类中自动获取泛型子类的实体类型？

**A：**
```java
public abstract class BaseService<T> {
    private final Class<T> entityClass;
    
    @SuppressWarnings("unchecked")
    protected BaseService() {
        ParameterizedType pt = (ParameterizedType) 
            getClass().getGenericSuperclass();
        this.entityClass = (Class<T>) pt.getActualTypeArguments()[0];
    }
}

// 子类
public class UserService extends BaseService<User> {
    // entityClass 自动为 User.class
}
```

这是 MyBatis-Plus、JPA 等框架中 BaseMapper/BaseRepository 获取实体类型的常用技巧。
