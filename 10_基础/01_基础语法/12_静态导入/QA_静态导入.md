---
title: static import 静态导入
tags:
  - Java/基础语法
  - 原理型
  - 问答
module: 01_基础语法
created: 2026-04-18
---

# static import 静态导入

## Q1：什么是 static import？有什么作用？

**A**：static import（静态导入）是 JDK 5 引入的语法，允许直接导入类或接口的静态成员（静态方法和静态字段），使用时无需再写类名限定。

```java
import static java.lang.Math.sqrt;

// 传统方式：Math.sqrt(25)
// static import 后：sqrt(25)
```

---

## Q2：static import 和普通 import 有什么区别？

**A**：

| 对比项 | `import` | `import static` |
|--|---|---|
| 导入目标 | 类或接口 | 类的静态成员（方法/字段） |
| 使用方式 | `Math.sqrt()` | `sqrt()` |
| 适用场景 | 引用类型 | 频繁使用的静态工具方法 |

---

## Q3：static import 有什么注意事项？

**A**：
1. **避免通配符导入**：`import static java.lang.Math.*;` 会导入所有静态成员，容易造成命名冲突，推荐精确导入指定成员
2. **不要滥用**：只在静态方法被频繁使用时才用（如 JUnit 的 `assertEquals`），否则保留类名可读性更好
3. **命名冲突**：不能同时导入两个类中同名的静态成员，需保留类名限定来解决

---

## Q4：static import 可以导入什么？

**A**：
- 静态方法：`import static java.util.Collections.sort;`
- 静态字段（常量）：`import static java.lang.Math.PI;`
- 静态内部类：`import static java.util.Map.Entry;`

不能导入：实例方法、实例字段、非静态内部类。

---

## Q5：实际开发中什么时候该用 static import？

**A**：典型适用场景：
- **单元测试**：`import static org.junit.Assert.*;` 简化断言
- **Math 运算**：`import static java.lang.Math.PI;` 减少重复
- **工具类方法**：`import static java.util.Collections.emptyList;`

核心判断标准：**去掉类名后，代码的可读性是否仍然清晰**。如果 `sort(list)` 比 `Collections.sort(list)` 更清晰且不会造成歧义，就值得用。

---

## 关联知识点
