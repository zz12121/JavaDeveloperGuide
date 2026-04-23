---
title: String 不可变性
tags:
  - Java/字符串
  - 原理型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String 不可变性

## Q：String 为什么是不可变的？

**A：** String 的不可变性由以下机制保证：
1. `String` 类被 `final` 修饰，不能被继承
2. 内部 `value` 数组被 `private final` 修饰，引用不可变且外部不可访问
3. 没有提供任何修改 `value` 内容的方法
4. 所有"修改"操作（substring、replace、concat）都是返回**新的 String 对象**

## Q：String 不可变有什么好处？

**A：**
1. **字符串常量池**：不可变对象可以安全共享，多个变量可以指向堆中的同一个 String 对象
2. **线程安全**：不可变对象天然线程安全，多线程下无需同步
3. **hashCode 缓存**：String 的 hashCode 计算后缓存，不变性保证 hash 始终一致，HashMap 查找高效
4. **安全性**：用作 HashMap 的 key、网络参数、文件路径时不会被意外修改

## Q：String 真的完全不可变吗？反射能修改吗？

**A：** 理论上可以通过反射修改 `value` 数组的内容，但这是**不推荐的危险操作**：
```java
Field field = String.class.getDeclaredField("value");
field.setAccessible(true);
char[] value = (char[]) field.get(str);
value[0] = 'H';  // 修改了底层数组
```
但修改后可能引发各种问题（字符串常量池中的值被污染、hashCode 不一致等），JDK 9+ 使用 byte[] 并增加了更多保护措施。
