
# new String 创建对象数

## Q1：new String("abc") 创建几个对象？

**A**：创建 **1 个或 2 个**对象：
- **1 个**：常量池已有 "abc"，只创建堆中的 String 对象
- **2 个**：常量池没有，先创建常量池对象，再创建堆对象

---

## Q2：String s = new String("a") + new String("b") 创建几个对象？

**A**：至少创建 **5-6 个对象**：
1. `new String("a")` - 堆对象
2. `"a"` - 常量池对象
3. `new String("b")` - 堆对象
4. `"b"` - 常量池对象
5. `StringBuilder` - 拼接用
6. `toString()` 产生的 "ab" 堆对象

---

## Q3：String s = "a" + "b" 创建几个对象？

**A**：创建 **0 个或 1 个**对象：
- **编译期优化**：`"a" + "b"` 在编译期优化为 `"ab"`
- 如果常量池已有 "ab"，不创建新对象
- 如果没有，创建 1 个常量池对象

---

## Q4：intern() 会创建新对象吗？

**A**：`intern()` 行为：
- **常量池已有**：返回常量池引用，不创建新对象
- **常量池没有**：将堆中对象的引用放入常量池，返回该引用
- JDK6：可能复制字符串到永久代
- JDK7+：放入堆中的常量池

---

## Q5：为什么 StringBuilder.toString() 要创建新对象？

**A**：`toString()` 源码：
```java
public String toString() {
    return new String(value, 0, count);
}
```
- StringBuilder 内部使用 char[]（非 final）
- toString() 必须创建新的 String 对象
- 因为 String 不可变，不能返回内部数组

---

## Q6：以下代码创建几个对象？

```java
String s = new String("hello") + "world";
```

**A**：创建 **4-5 个对象**：
1. `new String("hello")` - 1-2 个对象
2. `"hello"` - 常量池（可能）
3. `"world"` - 常量池（可能）
4. `StringBuilder` - 1 个
5. `toString()` - 1 个新 String

---

## Q7：如何减少 String 对象创建？

**A**：
1. **使用字面量**：`"abc"` 优先于 `new String("abc")`
2. **intern()**：复用已有字符串
3. **StringBuilder**：避免循环拼接
4. **String.format()**：适度使用
