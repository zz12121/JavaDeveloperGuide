---
title: String、StringBuilder、StringBuffer 区别
tags:
  - Java/字符串
  - 对比型
  - 问答
module: 09_字符串
created: 2026-04-18
---

# String、StringBuilder、StringBuffer 区别

## Q：String、StringBuilder、StringBuffer 有什么区别？

**A：**

| 对比项 | String | StringBuilder | StringBuffer |
|--------|--------|---------------|-------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 天然安全 | ❌ 不安全 | ✅ 安全（synchronized） |
| 性能 | 拼接最差 | 最快 | 略慢于 StringBuilder |
单线程拼接用 `StringBuilder`，多线程共享拼接用 `StringBuffer`，少量操作直接用 `String`。

## Q：为什么 StringBuilder 比 String 拼接快？

**A：** String 不可变，每次 `+` 拼接都会创建新的 String 对象（涉及数组拷贝）。StringBuilder 内部维护一个可变的 `char[]` 数组，`append()` 直接在原数组上追加字符，扩容时才创建新数组，避免了大量中间对象的创建。

## Q：StringBuffer 的线程安全是怎么实现的？

**A：** StringBuffer 的 `append()`、`insert()`、`delete()` 等方法都加了 `synchronized` 关键字，保证同一时刻只有一个线程操作内部数组。但锁机制带来性能开销，所以单线程场景推荐 StringBuilder。

## Q：String 的 + 拼接和 StringBuilder 性能一样吗？

**A：** 不完全一样。Java 编译器会对 `String +` 做**编译期优化**：如果 + 的两边都是字面量，直接合并为一个字符串。如果包含变量，编译器会自动用 StringBuilder 替代。但**循环内的 + 拼接**每次循环都会创建新的 StringBuilder，性能很差。
