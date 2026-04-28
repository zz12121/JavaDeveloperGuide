
# 字符串拼接优化

## Q1：Java 字符串拼接有哪些优化？

**A**：主要有两种优化：
1. **编译期常量折叠**：`"a" + "b"` → `"ab"`
2. **运行时 StringBuilder**：变量拼接使用 StringBuilder

---

## Q2：什么是编译期常量折叠？

**A**：编译期常量折叠是一种编译器优化：
- 当拼接的两侧都是**编译期常量**时
- 编译器直接计算结果，生成一个字面量
- 不需要运行时 StringBuilder

```java
// 源码
String s = "a" + "b" + "c";

// 编译后
String s = "abc";
```

---

## Q3：变量拼接一定用 StringBuilder 吗？

**A**：**不一定**：
- **JDK4**：使用 `StringBuffer`（synchronized）
- **JDK5+**：使用 `StringBuilder`（非同步）
- **JIT 优化**：某些情况下 JIT 可能进一步优化

---

## Q4：循环拼接应该注意什么？

**A**：循环拼接要**避免 `+=`**：
```java
// 错误：每次循环都创建新的 StringBuilder
for (int i = 0; i < 100; i++) {
    result += "a";  // 100 次！
}

// 正确：循环外创建 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append("a");
}
String result = sb.toString();
```

---

## Q5：StringBuilder 和 StringBuffer 的区别？

**A**：

| 区别 | StringBuilder | StringBuffer |
|------|---------------|--------------|
| **线程安全** | 不安全 | 安全（synchronized） |
| **性能** | 更高 | 较低 |
| **JDK 版本** | JDK5+ | JDK1.0 |
| **使用场景** | 单线程 | 多线程 |

---

## Q6：concat() 和 + 拼接的区别？

**A**：

| 区别 | `+` 拼接 | `concat()` |
|------|----------|-----------|
| **原理** | StringBuilder | 创建新 String |
| **常量折叠** | 支持 | 不支持 |
| **空指针** | 自动转字符串 | NPE |

```java
// concat() 示例
String s = "a".concat("b");  // "ab"
String s2 = null.concat("b");  // NPE
```

---

## Q7：以下代码创建几个对象？

```java
String s = "a" + "b" + "c";
```

**A**：**0 个或 1 个**：
- 编译期优化为 `"abc"`
- 如果常量池已有，不创建新对象
- 如果没有，创建 1 个常量池对象
