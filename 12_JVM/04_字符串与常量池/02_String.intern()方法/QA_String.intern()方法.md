
# String.intern() 方法

## Q1：intern() 方法的作用？

**A**：`String.intern()` 将字符串放入字符串常量池：
- 如果常量池已有该字符串，返回常量池中的引用
- 如果没有，将当前字符串加入常量池，返回常量池引用
- 目的是复用字符串对象，节省内存

---

## Q2：JDK6 和 JDK7+ intern() 的区别？

**A**：

| JDK 版本 | 常量池位置 | intern() 行为 |
|----------|------------|---------------|
| **JDK6** | 永久代 | 将字符串**复制**到永久代常量池 |
| **JDK7+** | 堆 | 将字符串**引用**放入常量池 |

---

## Q3：new String("ab") 创建几个对象？

**A**：
- **JDK6**：创建 2 个对象
  1. 字符串常量池中的 "ab"
  2. 堆中的 String 对象
- **JDK7+**：通常还是 2 个（字符串常量池在堆中）

---

## Q4：这段代码输出什么？

```java
String s1 = new String("a") + new String("b");
s1.intern();
String s2 = "ab";
System.out.println(s1 == s2);
```

**A**：
- **JDK7+**：true（intern 存储引用，s1 就是 "ab"）
- **JDK6**：false（s1 在堆中，"ab" 在永久代）

---

## Q5：字符串拼接什么情况会被优化？

**A**：
- **编译期优化**：`"a" + "b"` → `"ab"`（常量折叠）
- **StringBuilder 优化**：变量拼接在 JDK5+ 使用 StringBuilder
- **intern() 优化**：将拼接结果调用 intern() 可保证复用
