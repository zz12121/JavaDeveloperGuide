---
title: "QA_反射操作数组、枚举与内部类"
module: 08_反射
created: 2026-04-25
---

# 面试题：反射操作数组、枚举与内部类

### Q1：如何使用反射动态创建和操作数组？

**参考答案：**

**创建数组（Array.newInstance）**：

```java
// 方式1：直接指定组件类型和长度
Object array = Array.newInstance(String.class, 5);

// 方式2：二维数组
Object matrix = Array.newInstance(int.class, 3, 4);

// 方式3：通过 Class.forName 动态指定类型
Class<?> componentType = Class.forName("java.lang.Integer");
Object array = Array.newInstance(componentType, 10);
```

**操作数组元素**：

```java
Object array = Array.newInstance(String.class, 3);

// 写入（index, value）
Array.set(array, 0, "Hello");
Array.set(array, 1, "World");

// 读取
String s = (String) Array.get(array, 0);  // "Hello"

// 获取长度
int len = Array.getLength(array);  // 3
```

**模拟 ArrayList 扩容**：

```java
Object oldArray = Array.newInstance(String.class, 2);
Array.set(oldArray, 0, "a");
Array.set(oldArray, 1, "b");

// 新数组 1.5 倍扩容
int newCapacity = (int)(Array.getLength(oldArray) * 1.5);
Object newArray = Array.newInstance(String.class, newCapacity);
System.arraycopy(oldArray, 0, newArray, 0, Array.getLength(oldArray));
```

---

### Q2：枚举的反射操作有什么特殊之处？

**参考答案：**

**枚举的本质**：编译后继承 `java.lang.Enum`，每个常量是 `public static final` 字段。

```java
public enum Status {
    PENDING,    // 编译后：public static final Status PENDING = new Status();
    APPROVED,
    REJECTED;
}
```

**反射获取枚举值**：

```java
Class<Status> clazz = Status.class;

// 获取所有枚举常量（推荐方式）
Status[] constants = clazz.getEnumConstants();
// [PENDING, APPROVED, REJECTED]

// 通过字符串创建枚举（反序列化常用）
Status s = Status.valueOf("PENDING");  // 等于 Status.PENDING

// 判断是否为枚举类
clazz.isEnum();  // true
```

**枚举的反序列化（防注入）**：

```java
// Jackson/Fastjson 内部实现
public static Status deserialize(String name) {
    try {
        return Status.valueOf(name);  // 如果 name 不在枚举中，抛异常
    } catch (IllegalArgumentException e) {
        return null;  // 安全处理，避免反序列化注入
    }
}
```

---

### Q3：如何通过反射实例化成员内部类？

**参考答案：**

**成员内部类的构造器特点**：第一个参数必须是外部类类型。

```java
class Outer {
    class Inner {
        void greet() { System.out.println("Hello"); }
    }
}
```

**标准方式（new）**：

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

**纯反射方式**：

```java
// 1. 获取外部类实例
Class<?> outerClass = Class.forName("com.example.Outer");
Object outerInstance = outerClass.newInstance();

// 2. 获取内部类构造器（第一个参数是外部类类型）
Class<?> innerClass = Class.forName("com.example.Outer$Inner");
Constructor<?> ctor = innerClass.getDeclaredConstructor(outerClass);
ctor.setAccessible(true);  // 私有构造器也要能访问

// 3. 实例化
Object innerInstance = ctor.newInstance(outerInstance);
```

**从内部类实例反向获取外部类实例**：

```java
// 成员内部类编译后会生成 synthetic field: Outer this$0
Field outerField = innerInstance.getClass().getDeclaredField("this$0");
outerField.setAccessible(true);
Object outerInstance = outerField.get(innerInstance);

// 更简洁：getEnclosingClass()
Class<?> outerClass = innerInstance.getClass().getEnclosingClass();
```

---

### Q4：getDeclaredClasses() 和 getClasses() 的区别是什么？

**参考答案：**

```java
class Outer {
    class Inner {}
    static class StaticInner {}
}

class Derived extends Outer {}
```

| 方法 | 返回什么 | 是否包含继承的 |
|------|--------|:---:|
| `Outer.class.getDeclaredClasses()` | `[Inner, StaticInner]` | ❌ 仅当前类 |
| `Outer.class.getClasses()` | `[]` | ❌ 仅 public 内部类 |
| `Derived.class.getDeclaredClasses()` | `[]` | ❌ 继承的不算 |
| `Derived.class.getClasses()` | `[Inner]` | ✅ 父类的（public） |

**应用场景**：

```java
// 1. 获取所有内部类（包括私有）
for (Class<?> c : outerClass.getDeclaredClasses()) {
    System.out.println(c.getSimpleName());
}

// 2. 检查当前类是否在内部类中被调用
Method enclosingMethod = obj.getClass().getEnclosingMethod();
// 用于判断是否为匿名内部类的回调
```

---

### Q5：如何在反射中安全地拷贝数组（处理基本类型和引用类型）？

**参考答案：**

```java
public static Object deepCopyArray(Object sourceArray) {
    if (!sourceArray.getClass().isArray()) {
        throw new IllegalArgumentException("不是数组类型");
    }

    int length = Array.getLength(sourceArray);
    Class<?> componentType = sourceArray.getClass().getComponentType();
    Object newArray = Array.newInstance(componentType, length);

    for (int i = 0; i < length; i++) {
        Object element = Array.get(sourceArray, i);

        if (element == null) {
            continue;
        }

        // 基本类型不需要递归拷贝（Array.get 返回的是包装类型或基本类型）
        if (!element.getClass().isArray()) {
            Array.set(newArray, i, element);
        } else {
            // 多维数组递归拷贝
            Array.set(newArray, i, deepCopyArray(element));
        }
    }

    return newArray;
}
```

**面试加分点**：
- `getComponentType()` 获取数组的组件类型（用于创建新数组）
- 基本类型数组（如 `int[]`）的 `componentType` 是 `int.class`
- `Array.set()` 会自动拆箱基本类型值
